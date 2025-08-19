本项目是一个语音唤醒项目，当前的实现是：
若是单通道音频输入则
main_2ch.cpp
    │
    └─ if(numChan==1)
        │
        └─ Process()
            │
            └─ ProcessInternal_(from_iva=false)
                │
                ├─ PrepareInput_()
                │   ├─ GetIsEchoScene() - 回声检测
                │   ├─ NoiseReductionFuc() - 降噪处理
                │   └─ CutSamples() - 音频切片
                │
                ├─ OnlineFbank特征提取
                ├─ AI编码器推理
                ├─ TransducerKeywordDecoder解码
                └─ E2EMainBody() E2E验证
并有e2e和kws的联合打分机制判断是否唤醒
若是双通道音频输入则
main_2ch.cpp
    │
    ├─ 读取WAV文件 (remove_wav_header)
    │   ├─ 检测通道数 numChan
    │   └─ 分离左右声道 in_short_L[], in_short_R[]
    │
    ├─ 创建KeywordSpotter实例
    ├─ KwsInit() - 加载所有AI模型
    │
    └─ if(numChan==2)
        │
        └─ ProcessMultiChannel()
            │
            ├─ if(config.enable_iva && num_channels >= 2)
            │   │
            │   ├─ **IVA配置缓存检查和管理**
            │   │   ├─ iva_config_cache_.matches() - 检查配置变化
            │   │   ├─ if(need_init) iva_processor_->Init() - 重新初始化
            │   │   └─ else iva_processor_->Reset() - 重置算法状态
            │   │
            │   ├─ **数据格式转换和归一化** short[][] → double[][]
            │   │   └─ double_val = short_val / 32768.0
            │   │
            │   ├─ IVA处理链
            │   │   ├─ STFT::Forward() - 时频变换
            │   │   ├─ AuxIVA::Process() - 源分离
            │   │   └─ STFT::Inverse() - 逆变换
            │   │
            │   ├─ **条件性保存中间文件** (仅当提供文件名时)
            │   │   ├─ if(!config.current_audio_filename.empty())
            │   │   ├─ SaveIvaSeparatedMergedWav()
            │   │   ├─ SaveIvaKwsInputWav()
            │   │   ├─ SaveIvaRawFiles()
            │   │   └─ SaveIvaReport()
            │   │
            │   ├─ **数据格式转换** double[] → short[] (带范围限制)
            │   │   └─ 限制范围 [-32768, 32767]
            │   │
            │   └─ ProcessInternal_(from_iva=true)
            │       │
            │       ├─ PrepareInput_() - **执行降噪处理**
            │       ├─ OnlineFbank特征提取
            │       ├─ AI编码器推理
            │       ├─ TransducerKeywordDecoder解码
            │       └─ E2EMainBody() E2E验证
            │
            └─ else (IVA未启用)
                └─ Process() - 使用第一通道
                
并有e2e和kws的联合打分机制判断是否唤醒
现在需要改动：
1，接收音频对象支持pcm格式
2.降低代码耦合度，kws和噪声去除回声检测等分为两个模块（函数），其他也包含e2e模块，iva模块等，kws模块需要集成起来
3.ProcessMultiChannel函数的流程改为
3.1先提取第一个通道作为ProcessInternal_的输入，走正常流程，e2e正常（直接使用原始的第一个通道作为输入）
3.2若未返回唤醒成功的结果则走双通道，iva，用分离出来的第一路源直接做kws（没有噪声去除回声等相关内容），e2e正常（直接使用原始的第一个通道作为输入），仍利用新kws score和e2e融合判断是否返回语音唤醒
4.扩展功能：分析探索这些模块有无可以像积木一样任意拼接的方法。可以采用合理的顺序对音频进行操作，比如改变各个模块调用顺序，可以方便的插入新模块等等
1.不允许使用表情
2.注释有人味，专业
3.只允许修改
git_master\src\iva
git_master\src\keyword-spotter.cpp
git_master\src\keyword-spotter.h
main_2ch.cpp
