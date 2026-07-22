---
title: CURP共识算法详解-01-实现计划
abbrlink: 56935
date: 2026-07-21 15:06:08
tags:
---
在上一节中，我们介绍了 CURP 的核心理论。这一节将讨论如何基于现有的 Raft 实现扩展为 CURP。我们会分析现有代码结构，对比 Raft 和 CURP 的差异，并制定详细的实现计划。

------

# 1 为什么要基于 Raft 扩展？

在开始实现之前，一个重要的问题是：**为什么不从头开始实现 CURP，而是基于 Raft 扩展？**

## 1.1 复用已有成果

我们已经实现了一个功能完整的 Raft 共识协议，包含：

- 完整的选举机制
- 日志复制和提交
- 快照和日志压缩
- 成员变更支持

这些组件在 CURP 中同样需要，直接复用可以：

- 减少开发工作量
- 避免重复踩坑
- 加快验证速度

## 1.2 渐进式开发

基于 Raft 扩展允许我们：

1. 先实现一个最小可用的 CURP 版本
2. 逐步添加优化和高级特性
3. 每个阶段都可以验证正确性

## 1.3 对比学习

通过对比 Raft 和 CURP 的实现，可以更深入理解：

- 两者的设计权衡
- CURP 优化的本质
- 分布式系统的核心挑战

------

# 2 现有代码分析

让我们先了解现有 Raft 实现的结构，这是扩展的基础。

## 2.1 项目结构

```
tiny-distributed-kv/
├── proto/                  # Protocol Buffer 定义
│   ├── raft.proto          # Raft RPC 定义
│   └── node.proto          # 节点间通信定义
│
├── include/                # 头文件
│   ├── raft/
│   │   └── raft.h          # Raft 核心状态机
│   ├── storage/
│   │   └── log_vec.h       # 日志存储
│   ├── utils/
│   │   └── timer.h         # 定时器
│   └── grpc/
│       └── grpc_server.h   # gRPC 服务框架
│
├── src/                    # 源文件
│   ├── raft/
│   │   ├── raft.cpp       # Raft 状态机实现
│   │   └── raft_rpc.cpp   # Raft RPC 服务实现
│   ├── storage/
│   │   └── log_vec.cpp    # 日志存储实现
│   ├── utils/
│   │   └── timer.cpp      # 定时器实现
│   └── grpc/
│       └── grpc_server.cpp # gRPC 服务实现
│
└── test/                   # 测试文件
    └── dtest_raft.cpp       # Raft 测试
```

## 2.2 核心组件分析

### RaftNode 状态机

```
// include/raft/raft.h
class RaftNode {
public:
    // 节点角色
    enum class RaftState {
        LEADER,
        FOLLOWER,
        CANDIDATE
    };
    
private:
    // 持久化状态
    RaftState role_;
    int current_term_;
    int voted_for_;
    LogVec log_;                    // 操作日志
    
    // Leader 状态
    std::vector<uint64_t> next_index_;   // 每个 follower 的下一个日志索引
    std::vector<uint64_t> match_index_;  // 每个 follower 已复制的最高索引
    
    // 提交状态
    uint64_t commit_index_;
    uint64_t last_applied_;
    
    // 定时器
    std::unique_ptr<Timer> election_timer_;   // 选举超时定时器
    std::unique_ptr<Timer> heartbeat_timer_;  // 心跳定时器
    
public:
    // 核心方法
    void Start();                    // 启动节点
    void Elect();                    // 发起选举
    void SendHeartBeats();           // 发送心跳
    void AppendEntries(/*...*/);     // 追加日志
    void RequestVote(/*...*/);        // 请求投票
    
private:
    void applyCommittedEntries();    // 应用已提交的日志
    void becomeLeader();             // 成为 Leader
    void becomeFollower();           // 成为 Follower
};
```

**关键观察：**

- `commit_index_` 跟踪已提交的日志索引
- `last_applied_` 跟踪已应用到状态机的日志索引
- Leader 维护 `next_index_` 和 `match_index_` 用于日志复制

### 日志存储 LogVec

```
// include/storage/log_vec.h
class LogVec {
public:
    struct Entry {
        uint64_t term;
        uint64_t index;
        std::vector<uint8_t> command;
    };
    
private:
    std::vector<Entry> entries_;
    std::string log_dir_;
    
public:
    void Append(const Entry& entry);
    Entry Get(uint64_t index);
    void Truncate(uint64_t from_index);
    void Persist();                  // 持久化到磁盘
    
    uint64_t LastLogIndex();
    uint64_t LastLogTerm();
};
```

**关键特性：**

- 支持追加、随机访问、截断
- 持久化到磁盘，崩溃后可恢复
- 提供日志索引和任期查询

### RPC 服务

```
// Raft RPC 定义 (proto/raft.proto)
service RaftService {
    rpc RequestVote(RequestVoteRequest) returns (RequestVoteReply);
    rpc AppendEntries(AppendEntriesRequest) returns (AppendEntriesReply);
    rpc InstallSnapshot(InstallSnapshotRequest) returns (InstallSnapshotReply);
}

// 实现类
class RaftServiceImpl final : public RaftService::Service {
private:
    std::shared_ptr<RaftNode> raft_node_;
    
public:
    grpc::Status RequestVote(
        grpc::ServerContext* context,
        const RequestVoteRequest* request,
        RequestVoteReply* reply) override;
    
    grpc::Status AppendEntries(
        grpc::ServerContext* context,
        const AppendEntriesRequest* request,
        AppendEntriesReply* reply) override;
};
```

## 2.3 Raft 写操作流程回顾

让我们回顾 Raft 的写操作流程，这对理解 CURP 的扩展至关重要：

```
时间线（Raft 写操作）:
──────────────────────────────────────────────────────────────→

Client                 Leader                   Follower 1    Follower 2
  │                      │                          │             │
  │─── 1. 写请求 ───────→│                          │             │
  │                      │                          │             │
  │                      │─── 2. AppendEntries ─────→│             │
  │                      │─────────────────────────→│             │
  │                      │                          │             │
  │                      │←── 3. ACK ───────────────│             │
  │                      │←─────────────────────────│             │
  │                      │                          │             │
  │                      │─── 4. 更新 commitIndex ─→│ (多数派确认) │
  │                      │                          │             │
  │                      │─── 5. 应用到状态机 ─────→│             │
  │                      │                          │             │
  │←─── 6. 响应 ─────────│                          │             │
```

**关键点：**

- 步骤 2-3 需要等待多数派确认（至少 1 RTT）
- 步骤 4 需要等待 Leader 自己的 AppendEntries 返回
- 步骤 6 响应客户端，总延迟 ≈ 2 RTT

------

# 3 Raft vs CURP 对比

现在让我们详细对比 Raft 和 CURP 的差异，理解需要做哪些修改。

## 3.1 架构对比

```
Raft 架构（容错 f=1）:
─────────────────────────────────────
┌────────┐
│ Client │
└────────┘
     ↓
┌────────┐     ┌──────────┐     ┌──────────┐
│ Leader │←───→│ Follower │←───→│ Follower │
└────────┘     └──────────┘     └──────────┘
    ↑              ↑                ↑
    └──────────────┴────────────────┘
           同步复制（2 RTT）


CURP 架构（容错 f=1）:
─────────────────────────────────────
┌────────┐
│ Client │
└────────┘
  ↓    ↓
  ↓    └──────→┌──────────┐
  ↓             │ Witness  │  ← 轻量级，只存请求
  ↓             └──────────┘
  ↓
┌────────┐     ┌──────────┐
│ Master │────→│  Backup  │  ← 完整数据副本
└────────┘     └──────────┘
    ↑                ↑
    └────────────────┘
       异步复制（不阻塞客户端）
```

## 3.2 角色对比

| Raft 角色 | CURP 角色 | 主要变化                     |
| --------- | --------- | ---------------------------- |
| Leader    | Master    | 增加推测执行、管理未同步操作 |
| Follower  | Backup    | 基本不变，接收异步同步       |
| -         | Witness   | **新增**，轻量级持久化存储   |
| Candidate | -         | 选举机制可复用               |

## 3.3 写路径对比

### Raft 写路径

```
// Raft 的写操作
Status RaftNode::Propose(const Command& cmd) {
    // 1. 追加到本地日志
    uint64_t index = log_.Append(cmd);
    
    // 2. 发送 AppendEntries 到所有 follower
    for (auto& follower : followers_) {
        SendAppendEntries(follower, index);
    }
    
    // 3. 等待多数派确认（阻塞！）
    while (!hasMajorityAck(index)) {
        // 等待...
    }
    
    // 4. 提交并应用
    commit_index_ = index;
    Apply(cmd);
    
    // 5. 响应客户端
    return Status::OK;
}
```

**关键问题：步骤 3 是阻塞的，必须等待多数派确认。**

### CURP 写路径

```
// CURP 的写操作（快速路径）
Status CurpMaster::Propose(const Command& cmd) {
    // 1. 检查可交换性
    if (!canSpeculativeExecute(cmd.key)) {
        // 走慢路径，类似 Raft
        return slowPathWrite(cmd);
    }
    
    // 2. 标记为未同步
    unsynced_keys_.insert(cmd.key);
    
    // 3. 推测执行（不等待 Backup！）
    Apply(cmd);
    
    // 4. 立即响应客户端（不等待 Backup！）
    // 客户端并行发送到 Witness，由客户端保证持久性
    
    return Status::OK;
}

// 异步同步（后台线程）
void CurpMaster::asyncSyncThread() {
    while (running_) {
        std::this_thread::sleep_for(kSyncInterval);
        
        // 批量同步未同步的操作到 Backup
        auto to_sync = getUnsyncedOperations();
        for (auto& backup : backups_) {
            SendSync(backup, to_sync);
        }
        
        // 同步成功后清理 Witness
        onSyncComplete(to_sync);
    }
}
```

**关键区别：不等待 Backup，立即响应客户端。**

------

# 4 实现计划详解

我们分四个阶段实现 CURP，每个阶段都是可验证的。

## Phase 1: Witness 实现

Witness 是 CURP 的核心新组件，优先级最高。

### 4.1.1 Witness 的职责

Witness 需要实现以下功能：

| 功能                | 说明                 | 复杂度         |
| ------------------- | -------------------- | -------------- |
| **record**          | 接收并存储客户端请求 | 核心           |
| **可交换性检查**    | 拒绝冲突请求         | 核心           |
| **gc**              | 清理已同步的请求     | 中等           |
| **getRecoveryData** | 恢复时返回所有请求   | 简单           |
| **持久化**          | 崩溃后数据不丢失     | 可选（教学版） |

### 4.1.2 数据结构设计

```
// include/curp/witness.h
#pragma once

#include <string>
#include <vector>
#include <unordered_map>
#include <mutex>
#include <chrono>

// Witness 存储的请求记录
struct WitnessRecord {
    uint64_t rpc_id;                      // RIFL 唯一标识
    std::string key;                       // 操作的 key
    std::vector<uint8_t> request_data;     // 序列化的请求
    std::chrono::steady_clock::time_point timestamp;  // 用于 GC
};

class Witness {
public:
    Witness(size_t max_records = 10000);
    
    // 核心接口：记录请求
    enum class RecordResult {
        ACCEPTED,                    // 接受
        REJECTED_NOT_COMMUTATIVE,    // 冲突
        REJECTED_NO_SPACE,           // 空间不足
        REJECTED_NOT_ACCEPTING       // 恢复中，不接受新请求
    };
    
    RecordResult record(
        const std::string& master_id,
        const std::string& key,
        uint64_t rpc_id,
        const std::vector<uint8_t>& request_data
    );
    
    // GC：清理已同步的请求
    void garbage_collect(
        const std::vector<std::pair<std::string, uint64_t>>& synced  // (key, rpc_id)
    );
    
    // 恢复接口
    std::vector<WitnessRecord> get_recovery_data();
    void stop_accepting();  // 进入恢复模式
    void reset();           // 恢复完成，重置状态
    
    // 状态查询
    bool is_accepting() const { return accepting_; }
    size_t record_count() const { return records_.size(); }
    
private:
    std::mutex mtx_;
    bool accepting_ = true;
    size_t max_records_;
    
    // 按 key 存储请求记录
    std::unordered_map<std::string, WitnessRecord> records_;
    
    // 辅助方法
    bool hasConflict(const std::string& key) const;
};
```

### 4.1.3 核心实现

```
// src/curp/witness.cpp
#include "curp/witness.h"

Witness::RecordResult Witness::record(
    const std::string& master_id,
    const std::string& key,
    uint64_t rpc_id,
    const std::vector<uint8_t>& request_data
) {
    std::lock_guard<std::mutex> lock(mtx_);
    
    // 检查是否在接受新请求
    if (!accepting_) {
        return RecordResult::REJECTED_NOT_ACCEPTING;
    }
    
    // 可交换性检查：key 是否已被记录？
    if (records_.find(key) != records_.end()) {
        return RecordResult::REJECTED_NOT_COMMUTATIVE;
    }
    
    // 空间检查
    if (records_.size() >= max_records_) {
        return RecordResult::REJECTED_NO_SPACE;
    }
    
    // 存储记录
    records_[key] = WitnessRecord{
        .rpc_id = rpc_id,
        .key = key,
        .request_data = request_data,
        .timestamp = std::chrono::steady_clock::now()
    };
    
    return RecordResult::ACCEPTED;
}

void Witness::garbage_collect(
    const std::vector<std::pair<std::string, uint64_t>>& synced
) {
    std::lock_guard<std::mutex> lock(mtx_);
    
    for (const auto& [key, rpc_id] : synced) {
        auto it = records_.find(key);
        if (it != records_.end() && it->second.rpc_id == rpc_id) {
            records_.erase(it);
        }
    }
}
```

### 4.1.4 RPC 定义

```
// proto/witness.proto
syntax = "proto3";

package witness;

service WitnessService {
    // 客户端调用：记录请求
    rpc Record(RecordRequest) returns (RecordReply);
    
    // Master 调用：垃圾回收
    rpc GarbageCollect(GCRequest) returns (GCReply);
    
    // Master 调用：恢复时获取数据
    rpc GetRecoveryData(RecoveryRequest) returns (RecoveryData);
    
    // Master 调用：停止接受新请求
    rpc Stop(StopRequest) returns (StopReply);
    
    // Master 调用：重置状态
    rpc Reset(ResetRequest) returns (ResetReply);
}

message RecordRequest {
    string master_id = 1;
    string key = 2;
    uint64 rpc_id = 3;
    bytes request_data = 4;
}

message RecordReply {
    bool accepted = 1;
    enum RejectReason {
        UNKNOWN = 0;
        NOT_COMMUTATIVE = 1;
        NO_SPACE = 2;
        NOT_ACCEPTING = 3;
    }
    RejectReason reason = 2;
}

message GCRequest {
    repeated SyncedEntry entries = 1;
}

message SyncedEntry {
    string key = 1;
    uint64 rpc_id = 2;
}

message RecoveryRequest {
    string master_id = 1;
}

message RecoveryData {
    repeated WitnessRecord records = 1;
}

message WitnessRecord {
    string key = 1;
    uint64 rpc_id = 2;
    bytes request_data = 3;
}
```

------

## Phase 2: CurpMaster 实现

CurpMaster 是 CURP 的核心，负责推测执行和异步同步。

### 4.2.1 继承 RaftNode

```
// include/curp/curp_master.h
#pragma once

#include "../raft/raft.h"
#include "witness.h"
#include <unordered_set>
#include <atomic>

class CurpMaster : public RaftNode {
public:
    // 工厂方法
    static std::shared_ptr<CurpMaster> Create(
        const std::vector<NodeConfig>& cluster_configs,
        const std::vector<NodeConfig>& witness_configs,
        const std::string& log_dir,
        uint64_t cur_node_id
    );
    
    // CURP 特有接口
    enum class WriteResult {
        SUCCESS_FAST,    // 快速路径成功（1 RTT）
        SUCCESS_SLOW,    // 慢路径成功（2 RTT）
        FAILED
    };
    
    WriteResult proposeUpdate(
        const std::string& key,
        const std::vector<uint8_t>& value
    );
    
    std::optional<std::vector<uint8_t>> read(const std::string& key);
    
    // 状态查询
    bool hasUnsyncedKey(const std::string& key) const;
    
private:
    CurpMaster(
        const std::vector<NodeConfig>& cluster_configs,
        const std::vector<NodeConfig>& witness_configs,
        const std::string& log_dir,
        uint64_t cur_node_id
    );
    
    // Witness 管理
    std::vector<NodeConfig> witness_configs_;
    std::vector<std::unique_ptr<WitnessServiceStub>> witness_stubs_;
    
    // 未同步操作管理
    std::unordered_set<std::string> unsynced_keys_;
    std::unordered_map<std::string, uint64_t> unsynced_rpc_ids_;  // key -> rpc_id
    mutable std::mutex unsynced_mtx_;
    
    // 异步同步
    std::thread sync_thread_;
    std::atomic<bool> running_{true};
    std::condition_variable sync_cv_;
    
    // 核心方法
    bool canSpeculativeExecute(const std::string& key);
    void markUnsynced(const std::string& key, uint64_t rpc_id);
    void removeFromUnsynced(const std::string& key);
    
    void asyncSyncThread();
    void syncToBackup();
    void garbageCollectWitnesses();
    
    // 慢路径
    WriteResult slowPathWrite(
        const std::string& key,
        const std::vector<uint8_t>& value
    );
};
```

### 4.2.2 写操作实现

```
// src/curp/curp_master.cpp
#include "curp/curp_master.h"

CurpMaster::WriteResult CurpMaster::proposeUpdate(
    const std::string& key,
    const std::vector<uint8_t>& value
) {
    // 检查是否可以推测执行
    if (!canSpeculativeExecute(key)) {
        // 走慢路径
        return slowPathWrite(key, value);
    }
    
    // 快速路径：推测执行
    uint64_t rpc_id = nextRpcId();
    
    // 标记为未同步
    markUnsynced(key, rpc_id);
    
    // 执行操作（应用到状态机）
    kv_store_.put(key, value);
    
    // 返回成功，不等待 Backup
    return WriteResult::SUCCESS_FAST;
}

bool CurpMaster::canSpeculativeExecute(const std::string& key) {
    std::lock_guard<std::mutex> lock(unsynced_mtx_);
    return unsynced_keys_.find(key) == unsynced_keys_.end();
}

void CurpMaster::markUnsynced(const std::string& key, uint64_t rpc_id) {
    std::lock_guard<std::mutex> lock(unsynced_mtx_);
    unsynced_keys_.insert(key);
    unsynced_rpc_ids_[key] = rpc_id;
}

CurpMaster::WriteResult CurpMaster::slowPathWrite(
    const std::string& key,
    const std::vector<uint8_t>& value
) {
    // 先同步所有未同步的操作
    syncToBackup();
    
    // 清理未同步状态
    {
        std::lock_guard<std::mutex> lock(unsynced_mtx_);
        unsynced_keys_.clear();
        unsynced_rpc_ids_.clear();
    }
    
    // 执行新操作
    kv_store_.put(key, value);
    
    // 同步新操作到 Backup
    syncToBackup();
    
    return WriteResult::SUCCESS_SLOW;
}
```

### 4.2.3 异步同步线程

```
void CurpMaster::asyncSyncThread() {
    while (running_) {
        // 等待同步间隔或被唤醒
        std::unique_lock<std::mutex> lock(unsynced_mtx_);
        sync_cv_.wait_for(lock, std::chrono::milliseconds(10));
        
        if (!running_) break;
        
        if (unsynced_keys_.empty()) continue;
        
        lock.unlock();
        
        // 执行同步
        syncToBackup();
    }
}

void CurpMaster::syncToBackup() {
    // 获取未同步操作
    std::vector<std::pair<std::string, uint64_t>> to_sync;
    {
        std::lock_guard<std::mutex> lock(unsynced_mtx_);
        for (const auto& key : unsynced_keys_) {
            to_sync.push_back({key, unsynced_rpc_ids_[key]});
        }
    }
    
    if (to_sync.empty()) return;
    
    // 发送到所有 Backup
    for (auto& backup : backups_) {
        // 使用 Raft 的 AppendEntries 机制
        SendAppendEntries(backup, to_sync);
    }
    
    // 等待多数派确认（使用 Raft 的机制）
    // ...
    
    // 同步成功，清理 Witness
    garbageCollectWitnesses(to_sync);
    
    // 清理本地状态
    {
        std::lock_guard<std::mutex> lock(unsynced_mtx_);
        for (const auto& [key, _] : to_sync) {
            unsynced_keys_.erase(key);
            unsynced_rpc_ids_.erase(key);
        }
    }
}
```

------

## Phase 3: CurpClient 实现

CurpClient 负责并行发送请求到 Master 和 Witness。

### 4.3.1 客户端设计

```
// include/curp/curp_client.h
#pragma once

#include <string>
#include <vector>
#include <memory>
#include <future>

struct CurpConfig {
    std::string master_addr;
    std::vector<std::string> witness_addrs;
};

class CurpClient {
public:
    CurpClient(const CurpConfig& config);
    ~CurpClient();
    
    enum class WriteResult {
        SUCCESS,         // 成功
        SLOW_SUCCESS,    // 慢路径成功
        FAILED
    };
    
    WriteResult write(
        const std::string& key,
        const std::vector<uint8_t>& value
    );
    
    std::optional<std::vector<uint8_t>> read(const std::string& key);
    
private:
    CurpConfig config_;
    std::atomic<uint64_t> next_rpc_id_{1};
    
    // RPC stubs
    std::unique_ptr<KvServiceStub> master_stub_;
    std::vector<std::unique_ptr<WitnessServiceStub>> witness_stubs_;
    
    // 辅助方法
    bool recordToAllWitnesses(
        const std::string& key,
        uint64_t rpc_id,
        const std::vector<uint8_t>& request_data
    );
};
```

### 4.3.2 并行写实现

```
// src/curp/curp_client.cpp
CurpClient::WriteResult CurpClient::write(
    const std::string& key,
    const std::vector<uint8_t>& value
) {
    uint64_t rpc_id = next_rpc_id_++;
    
    // 序列化请求
    auto request_data = serializeRequest(key, value);
    
    // 并行发送：Master 和 Witness
    auto master_future = std::async(std::launch::async, [&]() {
        return sendToMaster(key, value, rpc_id);
    });
    
    auto witness_future = std::async(std::launch::async, [&]() {
        return recordToAllWitnesses(key, rpc_id, request_data);
    });
    
    // 等待结果
    auto master_reply = master_future.get();
    bool all_witness_accepted = witness_future.get();
    
    // 判断结果
    if (master_reply.success) {
        if (all_witness_accepted && !master_reply.need_sync) {
            return WriteResult::SUCCESS;  // 快速路径
        } else {
            return WriteResult::SLOW_SUCCESS;  // 慢路径
        }
    }
    
    return WriteResult::FAILED;
}

bool CurpClient::recordToAllWitnesses(
    const std::string& key,
    uint64_t rpc_id,
    const std::vector<uint8_t>& request_data
) {
    std::vector<std::future<bool>> futures;
    
    for (auto& stub : witness_stubs_) {
        futures.push_back(std::async(std::launch::async, [&]() {
            RecordRequest request;
            request.set_key(key);
            request.set_rpc_id(rpc_id);
            request.set_request_data(request_data.data(), request_data.size());
            
            auto reply = stub->Record(request);
            return reply.accepted();
        }));
    }
    
    // 等待所有 Witness 响应
    for (auto& f : futures) {
        if (!f.get()) return false;
    }
    
    return true;
}
```

------

## Phase 4: 恢复机制

### 4.4.1 恢复流程

```
void CurpMaster::recover() {
    spdlog::info("Starting CURP recovery...");
    
    // Phase 1: 从 Backup 恢复有序数据（复用 Raft）
    spdlog::info("Phase 1: Recovering from Backup...");
    recoverFromBackup();  // 使用 Raft 的恢复机制
    
    // Phase 2: 从 Witness 重放无序请求
    spdlog::info("Phase 2: Replaying from Witnesses...");
    for (auto& stub : witness_stubs_) {
        RecoveryRequest request;
        auto data = stub->GetRecoveryData(request);
        
        for (const auto& record : data.records()) {
            // 检查是否已执行（RIFL）
            if (!alreadyExecuted(record.rpc_id())) {
                replayRequest(record);
                executed_rpc_ids_.insert(record.rpc_id());
            }
        }
        
        break;  // 只需要一个 Witness
    }
    
    // Phase 3: 同步到 Backup
    spdlog::info("Phase 3: Syncing to Backup...");
    syncToBackup();
    
    // Phase 4: 重置所有 Witness
    spdlog::info("Phase 4: Resetting Witnesses...");
    for (auto& stub : witness_stubs_) {
        stub->Reset(ResetRequest());
    }
    
    spdlog::info("CURP recovery completed.");
}
```

------

# 5 文件结构规划

```
tiny-distributed-kv/
├── proto/
│   ├── raft.proto              # 已有
│   ├── node.proto              # 已有
│   └── witness.proto           # 新增
│
├── include/
│   ├── raft/                   # 已有，保持不变
│   ├── curp/
│   │   ├── witness.h           # 新增
│   │   ├── curp_master.h       # 新增
│   │   └── curp_client.h       # 新增
│   └── grpc/
│       ├── grpc_server.h       # 已有
│       └── witness_service_impl.h  # 新增
│
├── src/
│   ├── raft/                   # 已有
│   ├── curp/
│   │   ├── witness.cpp         # 新增
│   │   ├── curp_master.cpp     # 新增
│   │   └── curp_client.cpp     # 新增
│   └── grpc/
│       ├── grpc_server.cpp     # 已有
│       └── witness_service_impl.cpp  # 新增
│
└── test/
    ├── dtest_curp.cpp          # 新增：CURP 集成测试
    └── dtest_witness.cpp       # 新增：Witness 单元测试
```

------

# 6 实现顺序建议

```
Phase 1: Witness（预计 2-3 天）
────────────────────────────────
├── Day 1: proto/witness.proto + Witness 类框架
├── Day 2: Witness 核心逻辑（record, gc）
└── Day 3: Witness RPC 服务 + 单元测试

Phase 2: CurpMaster（预计 3-4 天）
────────────────────────────────
├── Day 1: CurpMaster 框架 + 推测执行
├── Day 2: 未同步操作管理
├── Day 3: 异步同步线程
└── Day 4: 集成测试

Phase 3: CurpClient（预计 1-2 天）
────────────────────────────────
├── Day 1: 客户端框架 + 并行发送
└── Day 2: 错误处理 + 测试

Phase 4: 恢复机制（预计 2-3 天）
────────────────────────────────
├── Day 1: 恢复流程框架
├── Day 2: RIFL 去重
└── Day 3: 恢复测试
```

------

# 7 教学版简化策略

作为教学版实现，我们可以做一些简化：

| 方面               | 完整实现         | 教学简化           | 影响                                 |
| ------------------ | ---------------- | ------------------ | ------------------------------------ |
| **RIFL**           | 完整的去重机制   | 简单的 RPC ID 集合 | 崩溃后可能重复执行（幂等操作无影响） |
| **Witness 数量**   | f 个独立 Witness | 单个 Witness       | 容错能力降低，但足够演示             |
| **批量同步**       | 自适应批大小     | 固定周期同步       | 性能不是最优                         |
| **GC**             | 复杂策略         | 简单超时清理       | 可能内存占用稍高                     |
| **Witness 持久化** | 必须持久化       | 内存存储           | 崩溃后可能丢失                       |
| **配置管理**       | 动态配置         | 静态配置           | 无法动态扩缩容                       |

**这些简化不影响正确性验证，但降低了实现复杂度。**

# 8 下一步

本节介绍了完整的实现计划。下一节将介绍 CURP 的测试框架设计，包括：

- 如何验证 CURP 的线性一致性
- 如何模拟网络故障和节点崩溃
- 如何测量快速路径成功率