# DSSM 音乐推荐流程

本流程描述用户数据、歌曲标签和歌曲文件的存储位置，以及 DSSM 粗排模型的离线训练、模型更新、离线评估和在线召回过程。

```mermaid
flowchart TB
    subgraph DATA["数据存储层"]
        direction LR
        USER_DATA["用户数据<br/>用户画像、播放历史、点击、收藏、点赞、跳过<br/><b>存储：GaussDB</b>"]
        ITEM_META["歌曲标签与元数据<br/>歌曲ID、歌手、类别、语言、版权、状态<br/><b>存储：GaussDB</b>"]
        SONG_FILE["歌曲文件<br/>音频、封面等对象数据<br/><b>存储：OBS</b>"]
    end

    subgraph OFFLINE["离线场景：训练、更新与发布"]
        direction TB
        ETL["1. 离线 ETL 与样本构建<br/>抽取用户行为和歌曲标签<br/>按时间构建训练集、验证集、测试集"]
        AUDIO_FEATURE{"是否使用音频内容特征？"}
        AUDIO_EXTRACT["从 OBS 读取歌曲文件<br/>提取音频标签或音频 Embedding"]
        TRAIN["2. 训练 DSSM 双塔<br/>用户特征 → 用户塔<br/>歌曲特征 → 物品塔<br/>Loss → 反向传播 → 更新权重"]
        SAVE_MODEL["3. 保存同一 model_version<br/>用户塔权重 + 物品塔权重<br/>特征编码器与 ID 映射"]
        ITEM_INFER["4. 使用新物品塔<br/>全量生成归一化 item embedding"]
        MILVUS_NEW["5. 写入新的 Milvus Collection<br/>item_id + embedding + model_version + 元数据<br/>建立 COSINE/IP ANN 索引"]
        RELEASE_CHECK{"离线评估是否达到发布门槛？"}
        RELEASE["6. 同步发布<br/>部署新用户塔<br/>原子切换 Milvus Alias"]
        REJECT["不发布<br/>调整数据、特征或参数后重新训练"]

        ETL --> AUDIO_FEATURE
        AUDIO_FEATURE -- "是" --> AUDIO_EXTRACT --> TRAIN
        AUDIO_FEATURE -- "否" --> TRAIN
        TRAIN --> SAVE_MODEL --> ITEM_INFER --> MILVUS_NEW --> RELEASE_CHECK
        RELEASE_CHECK -- "通过" --> RELEASE
        RELEASE_CHECK -- "未通过" --> REJECT --> ETL
    end

    subgraph EVAL["离线效果评估"]
        direction LR
        TEST_USER["测试用户历史行为<br/>作为用户塔输入"]
        GROUND_TRUTH["测试窗口内未来行为<br/>作为 ground truth"]
        TEST_EMB["待发布用户塔<br/>生成 user embedding"]
        TEST_RECALL["查询待发布 Milvus 索引<br/>召回 TopK"]
        METRICS["topk_metrics<br/>Recall@K / NDCG@K<br/>MRR@K / Precision@K"]

        TEST_USER --> TEST_EMB --> TEST_RECALL --> METRICS
        GROUND_TRUTH --> METRICS
    end

    subgraph ONLINE["在线场景：DSSM 粗排召回"]
        direction TB
        REQUEST["1. 用户发起推荐请求<br/>user_id + 场景上下文"]
        ONLINE_FEATURE["2. 获取最新用户特征<br/>GaussDB / 缓存 / 实时特征服务"]
        USER_INFER["3. 同版本用户塔在线推理<br/>实时生成归一化 user embedding"]
        MILVUS_SEARCH["4. 查询同版本 Milvus 索引<br/>按 COSINE/IP 召回 TopN 歌曲"]
        DOWNSTREAM["输出粗排候选集<br/>item_id + similarity score + model_version<br/>交给下游模块"]

        REQUEST --> ONLINE_FEATURE --> USER_INFER --> MILVUS_SEARCH --> DOWNSTREAM
    end

    USER_DATA --> ETL
    ITEM_META --> ETL
    SONG_FILE -. "仅在使用音频内容特征时" .-> AUDIO_EXTRACT

    USER_DATA --> TEST_USER
    USER_DATA --> GROUND_TRUTH
    MILVUS_NEW --> TEST_RECALL
    METRICS --> RELEASE_CHECK

    USER_DATA --> ONLINE_FEATURE
    RELEASE --> USER_INFER
    RELEASE --> MILVUS_SEARCH
```

## 核心原则

1. Milvus 存储的是歌曲 `item embedding` 及其元数据，不存储 OBS 中的歌曲文件。
2. 用户 `user embedding` 在线实时生成，也可以按 `user_id` 缓存，但通常不需要建立用户向量索引。
3. 用户塔与 Milvus 中的物品向量必须来自同一个 `model_version`。
4. 即使模型结构不变，只要物品塔、共享 Embedding 或相关权重发生变化，就需要全量刷新 `item embedding` 并重建索引。
5. `topk_metrics` 仅用于离线召回效果评估，不参与模型训练，也不在在线粗排请求中实时计算。
6. 在线 DSSM 服务到输出 TopN 粗排候选集为止，过滤、精排和最终用户响应属于下游流程。
