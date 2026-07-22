---
title: CURP共识算法详解-02-测试框架设计
abbrlink: 51394
date: 2026-07-21 15:08:46
tags: 技术
---
在实现了 CURP 共识协议之后，一个关键问题是：**如何验证实现的正确性？** 分布式系统的测试比单机系统复杂得多，需要考虑网络延迟、节点崩溃、消息丢失等各种故障场景。本节将详细介绍 CURP 的测试框架设计。

------

# 1 为什么分布式系统测试这么难？

在讨论测试框架之前，让我们先理解分布式系统测试的挑战。

## 1.1 非确定性性行为

单机程序的执行通常是确定性的：给定相同的输入，输出也是相同的。但分布式系统不同：

```
场景：两个客户端同时写入相同 key

时间线 1（Client A 先到达）:
Client A: SET key = value1 → Master → Backup → 成功
Client B: SET key = value2 → Master → Backup → 成功
最终结果: key = value2

时间线 2（Client B 先到达）:
Client B: SET key = value2 → Master → Backup → 成功
Client A: SET key = value1 → Master → Backup → 成功
最终结果: key = value1

两种情况都是正确的，但结果不同！
```

**挑战：** 测试必须接受所有正确的执行顺序，不能只验证一种预期结果。

## 1.2 故障注入的复杂性

分布式系统必须正确处理各种故障：

| 故障类型     | 表现             | 测试难点         |
| ------------ | ---------------- | ---------------- |
| **网络延迟** | 消息延迟到达     | 可能打乱消息顺序 |
| **网络分区** | 节点间无法通信   | 可能导致脑裂     |
| **消息丢失** | 消息永远不会到达 | 需要重传机制     |
| **节点崩溃** | 进程突然终止     | 需要恢复机制     |
| **崩溃恢复** | 节点重启后恢复   | 状态可能不一致   |

**挑战：** 这些故障可能以任意组合、任意顺序发生，穷举测试是不可能的。

## 1.3 并发问题难以复现

分布式系统的 bug 通常与特定的执行顺序相关：

```
// 一个典型的竞态条件
void CurpMaster::proposeUpdate(const std::string& key, ...) {
    // 检查可交换性
    if (unsynced_keys_.count(key) > 0) {
        // 走慢路径
    }
    
    // ⚠️ 这里有竞态条件！
    // 另一个线程可能在这里修改了 unsynced_keys_
    
    unsynced_keys_.insert(key);  // 可能覆盖其他线程的修改
}
```

**挑战：** 这类 bug 在正常测试中可能永远不会触发，只有在特定的并发场景下才会出现。

------

# 2 测试策略总览

针对上述挑战，我们采用多层测试策略：

```
测试金字塔（从下到上）:
─────────────────────────────────────────────
│          端到端测试（E2E Tests）             │  ← 真实网络环境
│              数量：少量                       │
├─────────────────────────────────────────────┤
│        集成测试（Integration Tests）         │  ← 多节点交互
│              数量：中等                       │
├─────────────────────────────────────────────┤
│          单元测试（Unit Tests）              │  ← 单个组件
│              数量：大量                       │
└─────────────────────────────────────────────┘
```

## 2.1 单元测试

**目标：** 验证单个组件的正确性。

**测试内容：**

- Witness 的可交换性检查逻辑
- CurpMaster 的推测执行条件判断
- CurpClient 的并行发送逻辑

**特点：**

- 快速执行
- 隔离环境
- 易于定位问题

## 2.2 集成测试

**目标：** 验证多个组件的交互正确性。

**测试内容：**

- Client → Master → Witness 的完整写路径
- Master → Backup 的异步同步
- Master 崩溃后的恢复流程

**特点：**

- 使用模拟网络
- 可控的故障注入
- 验证组件协作

## 2.3 端到端测试

**目标：** 验证真实环境下的正确性。

**测试内容：**

- 长时间运行下的稳定性
- 真实网络延迟下的性能
- 各种故障场景的恢复

**特点：**

- 使用真实网络
- 长时间运行
- 接近生产环境

------

# 3 测试框架设计

为了支持上述测试策略，我们需要一个完整的测试框架。

## 3.1 框架架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Test Controller                         │
│  (控制测试流程、收集结果、验证正确性)                          │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Network       │    │ Node          │    │ Consistency   │
│ Simulator     │    │ Controller    │    │ Checker       │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Simulated Cluster                        │
│  ┌─────┐  ┌─────┐  ┌─────┐    ┌─────┐  ┌─────┐            │
│  │Master│  │Backup│  │Backup│    │Witness│×f            │
│  └─────┘  └─────┘  └─────┘    └─────┘                      │
└─────────────────────────────────────────────────────────────┘
```

## 3.2 核心组件

### Network Simulator（网络模拟器）

**职责：** 模拟网络行为，包括延迟、丢包、分区。

```
class NetworkSimulator {
public:
    // 设置节点间延迟
    void setDelay(int from_node, int to_node, int delay_ms);
    
    // 设置丢包率
    void setPacketLoss(double probability);
    
    // 模拟网络分区
    void partition(const std::vector<int>& group1, 
                   const std::vector<int>& group2);
    
    // 恢复网络
    void heal();
    
    // 发送消息（会根据配置添加延迟或丢弃）
    void sendMessage(int from, int to, const Message& msg);
    
    // 处理所有待处理的消息（推进时间）
    void processMessages();
    
private:
    struct DelayedMessage {
        int from;
        int to;
        Message msg;
        int delivery_time;  // 送达时间
    };
    
    std::vector<DelayedMessage> pending_messages_;
    std::map<std::pair<int,int>, int> delays_;
    double packet_loss_prob_ = 0.0;
    std::set<std::pair<int,int>> blocked_links_;
    int current_time_ = 0;
};
```

**使用示例：**

```
TEST(CurpTest, NetworkLatency) {
    NetworkSimulator net;
    
    // 设置 Master 和 Backup 之间有 50ms 延迟
    net.setDelay(MASTER, BACKUP1, 50);
    
    // 发送消息不会立即到达
    net.sendMessage(MASTER, BACKUP1, AppendEntriesRequest{...});
    
    // 此时 Backup 还没收到消息
    EXPECT_FALSE(backup1->hasReceivedMessage());
    
    // 推进时间 50ms
    net.processMessages();
    
    // 现在 Backup 收到了
    EXPECT_TRUE(backup1->hasReceivedMessage());
}
```

### Node Controller（节点控制器）

**职责：** 控制节点的生命周期，模拟崩溃和恢复。

```
class NodeController {
public:
    // 启动节点
    void startNode(int node_id, const NodeConfig& config);
    
    // 停止节点（优雅关闭）
    void stopNode(int node_id);
    
    // 崩溃节点（模拟突然断电）
    void crashNode(int node_id);
    
    // 恢复节点（从持久化状态恢复）
    void recoverNode(int node_id);
    
    // 检查节点是否运行
    bool isRunning(int node_id) const;
    
    // 获取节点状态
    NodeState getState(int node_id) const;
    
private:
    std::map<int, std::unique_ptr<CurpNode>> nodes_;
    std::map<int, bool> running_;
    std::map<int, NodeConfig> configs_;
};
```

**关键区别：**

- `stopNode`：优雅关闭，数据完整保存
- `crashNode`：模拟崩溃，内存数据丢失，只有持久化的数据能恢复

```
TEST(CurpTest, CrashRecovery) {
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(BACKUP1, backup_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    // 写入数据
    client.write("key1", "value1");
    
    // Master 崩溃（内存数据丢失）
    nc.crashNode(MASTER);
    
    // 恢复 Master
    nc.recoverNode(MASTER);
    
    // 验证数据恢复
    auto value = client.read("key1");
    EXPECT_EQ(value, "value1");  // 数据应该完整恢复
}
```

### Consistency Checker（一致性检查器）

**职责：** 验证操作的线性一致性。

```
class ConsistencyChecker {
public:
    // 记录写操作
    void recordWrite(
        int client_id,
        const std::string& key,
        const std::vector<uint8_t>& value,
        uint64_t start_time,
        uint64_t end_time
    );
    
    // 记录读操作
    void recordRead(
        int client_id,
        const std::string& key,
        const std::vector<uint8_t>& value,
        uint64_t start_time,
        uint64_t end_time
    );
    
    // 检查线性一致性
    bool checkLinearizability();
    
    // 检查数据完整性
    bool checkDataIntegrity();
    
private:
    struct OperationRecord {
        enum Type { READ, WRITE };
        Type type;
        int client_id;
        std::string key;
        std::vector<uint8_t> value;
        uint64_t start_time;
        uint64_t end_time;
    };
    
    std::vector<OperationRecord> operations_;
    
    // 线性一致性检查算法
    bool checkLinearizabilityInternal(
        const std::vector<OperationRecord>& ops
    );
    
    // 构建实时序图
    Graph buildRealTimeGraph(const std::vector<OperationRecord>& ops);
    
    // 检测是否存在环
    bool hasCycle(const Graph& graph);
};
```

## 3.3 线性一致性检查原理

线性一致性（Linearizability）是最强的一致性模型，要求：

> 所有操作看起来都在某个时间点原子执行，且操作的顺序与实时序一致。

### 检查方法

我们使用 **Wing & Gong 算法** 的简化版本：

1. **记录所有操作**：每个操作记录开始时间和结束时间
2. **构建实时序图**：
   - 如果操作 A 的结束时间 < 操作 B 的开始时间，则 A → B
   - 如果操作 A 和 B 作用于相同 key，且至少一个是写操作，则它们之间有顺序约束
3. **检测环**：如果图中存在环，则违反线性一致性

```
bool ConsistencyChecker::checkLinearizability() {
    // 构建实时序图
    Graph graph = buildRealTimeGraph(operations_);
    
    // 检测是否存在环
    if (hasCycle(graph)) {
        return false;  // 存在环，违反线性一致性
    }
    
    return true;  // 线性一致
}

Graph ConsistencyChecker::buildRealTimeGraph(
    const std::vector<OperationRecord>& ops
) {
    Graph graph(ops.size());
    
    for (size_t i = 0; i < ops.size(); i++) {
        for (size_t j = i + 1; j < ops.size(); j++) {
            // 实时序约束
            if (ops[i].end_time < ops[j].start_time) {
                graph.addEdge(i, j);  // i 必须在 j 之前
            }
            
            // 相同 key 的顺序约束
            if (ops[i].key == ops[j].key) {
                // 如果至少有一个是写操作，则需要排序
                if (ops[i].type == OperationRecord::WRITE ||
                    ops[j].type == OperationRecord::WRITE) {
                    // 需要确定顺序...
                    graph.addEdge(i, j);  // 或 j → i
                }
            }
        }
    }
    
    return graph;
}
```

------

# 4 测试场景设计

## 4.1 单元测试

### Witness 单元测试

```
// test/dtest_witness.cpp

// 测试：可交换性检查 - 不同 key
TEST(WitnessTest, AcceptCommutativeOps) {
    Witness w;
    
    // 第一次写入 key1
    auto r1 = w.record("master1", "key1", 1, {});
    EXPECT_EQ(r1, Witness::RecordResult::ACCEPTED);
    
    // 第二次写入 key2（不同 key，可交换）
    auto r2 = w.record("master1", "key2", 2, {});
    EXPECT_EQ(r2, Witness::RecordResult::ACCEPTED);
    
    EXPECT_EQ(w.record_count(), 2);
}

// 测试：可交换性检查 - 相同 key
TEST(WitnessTest, RejectNonCommutativeOps) {
    Witness w;
    
    // 第一次写入 key1
    w.record("master1", "key1", 1, {});
    
    // 第二次写入 key1（相同 key，不可交换）
    auto r = w.record("master1", "key1", 2, {});
    EXPECT_EQ(r, Witness::RecordResult::REJECTED_NOT_COMMUTATIVE);
}

// 测试：空间限制
TEST(WitnessTest, RejectWhenFull) {
    Witness w(3);  // 最多存储 3 个请求
    
    w.record("master1", "key1", 1, {});
    w.record("master1", "key2", 2, {});
    w.record("master1", "key3", 3, {});
    
    // 第 4 个请求应该被拒绝
    auto r = w.record("master1", "key4", 4, {});
    EXPECT_EQ(r, Witness::RecordResult::REJECTED_NO_SPACE);
}

// 测试：垃圾回收
TEST(WitnessTest, GarbageCollect) {
    Witness w;
    
    w.record("master1", "key1", 1, {});
    w.record("master1", "key2", 2, {});
    w.record("master1", "key3", 3, {});
    
    EXPECT_EQ(w.record_count(), 3);
    
    // GC 清理 key1 和 key2
    w.garbage_collect({
        {"key1", 1},
        {"key2", 2}
    });
    
    EXPECT_EQ(w.record_count(), 1);
    
    // key1 可以重新接受
    auto r = w.record("master1", "key1", 4, {});
    EXPECT_EQ(r, Witness::RecordResult::ACCEPTED);
}

// 测试：恢复模式
TEST(WitnessTest, StopAccepting) {
    Witness w;
    
    w.record("master1", "key1", 1, {});
    
    // 进入恢复模式
    w.stop_accepting();
    
    // 不接受新请求
    auto r = w.record("master1", "key2", 2, {});
    EXPECT_EQ(r, Witness::RecordResult::REJECTED_NOT_ACCEPTING);
    
    // 但可以获取恢复数据
    auto data = w.get_recovery_data();
    EXPECT_EQ(data.size(), 1);
    EXPECT_EQ(data[0].key, "key1");
}
```

### CurpMaster 单元测试

```
// test/dtest_curp_master.cpp

// 测试：推测执行条件
TEST(CurpMasterTest, CanSpeculativeExecute) {
    CurpMaster master(...);
    
    // 初始状态，可以推测执行
    EXPECT_TRUE(master.canSpeculativeExecute("key1"));
    
    // 写入 key1，标记为未同步
    master.proposeUpdate("key1", "value1");
    
    // key1 已未同步，不能再次推测执行
    EXPECT_FALSE(master.canSpeculativeExecute("key1"));
    
    // key2 未被影响，可以推测执行
    EXPECT_TRUE(master.canSpeculativeExecute("key2"));
}

// 测试：快速路径写操作
TEST(CurpMasterTest, FastPathWrite) {
    CurpMaster master(...);
    MockWitness witness;
    master.addWitness(&witness);
    
    // 写入操作
    auto result = master.proposeUpdate("key1", "value1");
    
    // 应该立即返回（不等待 Backup）
    EXPECT_EQ(result, CurpMaster::WriteResult::SUCCESS_FAST);
    
    // 数据应该已经写入
    auto value = master.read("key1");
    EXPECT_TRUE(value.has_value());
    EXPECT_EQ(*value, "value1");
}
```

## 4.2 集成测试

### 完整写路径测试

```
// test/dtest_curp_integration.cpp

TEST(CurpIntegrationTest, FullWritePath) {
    // 设置集群
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(BACKUP1, backup_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    // 写入数据
    auto result = client.write("key1", "value1");
    EXPECT_EQ(result, CurpClient::WriteResult::SUCCESS);
    
    // 从 Master 读取
    auto value = client.read("key1");
    EXPECT_TRUE(value.has_value());
    EXPECT_EQ(*value, "value1");
    
    // 等待异步同步完成
    std::this_thread::sleep_for(100ms);
    
    // 从 Backup 读取（验证同步成功）
    auto backup_value = readFromBackup(BACKUP1, "key1");
    EXPECT_EQ(backup_value, "value1");
}

// 测试：Master 崩溃恢复
TEST(CurpIntegrationTest, MasterCrashRecovery) {
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(BACKUP1, backup_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    // 写入多条数据
    client.write("key1", "value1");
    client.write("key2", "value2");
    client.write("key3", "value3");
    
    // Master 崩溃
    nc.crashNode(MASTER);
    
    // 恢复 Master
    nc.recoverNode(MASTER);
    
    // 验证所有数据都恢复了
    EXPECT_EQ(client.read("key1"), "value1");
    EXPECT_EQ(client.read("key2"), "value2");
    EXPECT_EQ(client.read("key3"), "value3");
}

// 测试：网络延迟下的写操作
TEST(CurpIntegrationTest, WriteWithNetworkDelay) {
    NetworkSimulator net;
    
    // 设置 50ms 网络延迟
    net.setDelay(CLIENT, MASTER, 50);
    net.setDelay(CLIENT, WITNESS1, 50);
    
    NodeController nc;
    nc.setNetworkSimulator(&net);
    nc.startNode(MASTER, master_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    auto start = std::chrono::steady_clock::now();
    auto result = client.write("key1", "value1");
    auto end = std::chrono::steady_clock::now();
    
    // 应该在约 1 RTT 内完成（~50ms），而不是 2 RTT（~100ms）
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    EXPECT_LT(duration.count(), 70);  // 留一些余量
}
```

## 4.3 故障注入测试

### 网络分区测试

```
TEST(CurpFaultTest, NetworkPartition) {
    NetworkSimulator net;
    NodeController nc;
    
    nc.startNode(MASTER, master_config);
    nc.startNode(BACKUP1, backup_config);
    nc.startNode(BACKUP2, backup_config);
    nc.startNode(WITNESS1, witness_config);
    nc.startNode(WITNESS2, witness_config);
    
    CurpClient client(client_config);
    
    // 正常写入
    client.write("key1", "value1");
    
    // 网络分区：Master 和 Witness1 分在一组
    net.partition({MASTER, WITNESS1}, {BACKUP1, BACKUP2, WITNESS2});
    
    // 此时写入仍然可以成功（通过 Witness1）
    auto result = client.write("key2", "value2");
    
    // 恢复网络
    net.heal();
    
    // 等待同步
    std::this_thread::sleep_for(100ms);
    
    // 所有节点数据应该一致
    // ...
}
```

### 并发写入测试

```
TEST(CurpConcurrencyTest, ConcurrentWrites) {
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(WITNESS1, witness_config);
    
    const int NUM_CLIENTS = 10;
    const int OPS_PER_CLIENT = 100;
    
    std::vector<std::thread> threads;
    std::atomic<int> success_count{0};
    
    for (int i = 0; i < NUM_CLIENTS; i++) {
        threads.emplace_back([&, i]() {
            CurpClient client(client_config);
            
            for (int j = 0; j < OPS_PER_CLIENT; j++) {
                std::string key = "key_" + std::to_string(i) + "_" + std::to_string(j);
                auto result = client.write(key, "value");
                if (result == CurpClient::WriteResult::SUCCESS) {
                    success_count++;
                }
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    // 大部分操作应该成功
    EXPECT_GT(success_count, NUM_CLIENTS * OPS_PER_CLIENT * 0.9);
}
```

------

# 5 性能测试

## 5.1 快速路径成功率

```
TEST(CurpPerformanceTest, FastPathSuccessRate) {
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    int total_ops = 10000;
    int fast_path_count = 0;
    
    for (int i = 0; i < total_ops; i++) {
        std::string key = "key_" + std::to_string(i);
        auto result = client.write(key, "value");
        
        if (result == CurpClient::WriteResult::SUCCESS) {
            fast_path_count++;
        }
    }
    
    double fast_path_rate = (double)fast_path_count / total_ops;
    
    // 快速路径成功率应该 > 90%（假设 key 不重复）
    EXPECT_GT(fast_path_rate, 0.90);
    
    std::cout << "Fast path rate: " << fast_path_rate * 100 << "%" << std::endl;
}
```

## 5.2 延迟测量

```
TEST(CurpPerformanceTest, WriteLatency) {
    NodeController nc;
    nc.startNode(MASTER, master_config);
    nc.startNode(WITNESS1, witness_config);
    
    CurpClient client(client_config);
    
    std::vector<long> latencies;
    
    for (int i = 0; i < 1000; i++) {
        std::string key = "key_" + std::to_string(i);
        
        auto start = std::chrono::steady_clock::now();
        client.write(key, "value");
        auto end = std::chrono::steady_clock::now();
        
        auto latency = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
        latencies.push_back(latency);
    }
    
    // 计算统计信息
    std::sort(latencies.begin(), latencies.end());
    
    long p50 = latencies[latencies.size() * 0.50];
    long p95 = latencies[latencies.size() * 0.95];
    long p99 = latencies[latencies.size() * 0.99];
    
    std::cout << "Write latency (µs):" << std::endl;
    std::cout << "  P50: " << p50 << std::endl;
    std::cout << "  P95: " << p95 << std::endl;
    std::cout << "  P99: " << p99 << std::endl;
}
```

------

# 6 测试指标

| 指标               | 说明               | 目标值              |
| ------------------ | ------------------ | ------------------- |
| **快速路径成功率** | 1 RTT 完成的比例   | > 90%               |
| **写延迟 P50**     | 中位数写延迟       | < 5ms（同数据中心） |
| **写延迟 P99**     | 99% 写延迟         | < 20ms              |
| **恢复时间**       | 崩溃后恢复时间     | < 1s                |
| **数据丢失率**     | 故障后数据丢失比例 | 0%                  |
| **线性一致性违反** | 检测到的不一致次数 | 0                   |

------

# 7 持续集成

建议使用 GitHub Actions 进行持续集成测试：

```
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y cmake g++ libgrpc++-dev
    
    - name: Build
      run: |
        mkdir build && cd build
        cmake ..
        make -j$(nproc)
    
    - name: Run unit tests
      run: |
        cd build
        ctest --output-on-failure
    
    - name: Run integration tests
      run: |
        cd build
        ./test/dtest_curp_integration
```

------

# 8 总结

分布式系统测试是一个复杂但必要的任务。通过本文介绍的测试框架：

1. **Network Simulator**：模拟网络行为，验证系统在各种网络条件下的正确性
2. **Node Controller**：控制节点生命周期，测试崩溃恢复
3. **Consistency Checker**：验证线性一致性

我们可以系统性地验证 CURP 实现的正确性和性能。

**测试不是一次性的工作，而是持续的过程。** 每次修改代码都应该运行完整的测试套件，确保没有引入新的 bug。

------

# 参考资料

- *Linearizability: A Correctness Condition for Concurrent Objects*, Herlihy & Wing, 1990
- *Testing Distributed Systems for Linearizability*, aphyr.com
- *Jepsen: Calls Out Distributed Systems on Their Claims*, jepsen.io