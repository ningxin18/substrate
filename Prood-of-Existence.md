第一课 课程大纲

## Proof of Existence 教程进阶

- 回顾开源协作流程
- 回顾存证教程
- Transaction Weight & Fees
- SCALE codec 介绍
- 单元测试
- FRAME 基础功能模块介绍
    - timestamp
    - system
    - utility
    - Transaction-payment

## 课后作业

编写作业，需要提交代码连接

- 第一题： 编写存证模块的单元测试代码，包括：
    - 创建存证的测试用例
    - 撤销存证的测试用例
    - 转移存证的测试用例
- 第二题: 创建存证时，为存证内容的哈希值Vec<u8>
    - 设置长度上限，超过限制时返回错误
        - Get < T > 的是使用
    - 并编写测试用例




## 基本回顾

- 区块链和Substrate基本知识
- Rust基本知识
- 如何开发runtime模块
    - 存储数据类型
    - 宏的使用
    - Polkadotjs/api

## 链上存证的介绍

存证是一种在线服务，可用于在某一时间点验证计算机文件的存在性，最早是通过比特币网络带有时间戳的交易实现的。存证的应用场景有：

- 数字版权
- 司法存证
- 供应链溯源
- 电子发票
- ....

