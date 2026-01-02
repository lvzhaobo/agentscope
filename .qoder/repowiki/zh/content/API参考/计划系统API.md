# 计划系统API

<cite>
**本文档中引用的文件**   
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py)
- [plan_model.py](file://src/agentscope/plan/_plan_model.py)
- [storage_base.py](file://src/agentscope/plan/_storage_base.py)
- [in_memory_storage.py](file://src/agentscope/plan/_in_memory_storage.py)
- [main_agent_managed_plan.py](file://examples/functionality/plan/main_agent_managed_plan.py)
- [main_manual_plan.py](file://examples/functionality/plan/main_manual_plan.py)
- [plan_test.py](file://tests/plan_test.py)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件](#核心组件)
3. [任务规划接口](#任务规划接口)
4. [计划存储后端](#计划存储后端)
5. [计划与智能体联动](#计划与智能体联动)
6. [多层级计划管理](#多层级计划管理)
7. [计划可视化与人工干预](#计划可视化与人工干预)
8. [异常恢复策略](#异常恢复策略)

## 简介
计划系统模块为智能体提供了完整的任务规划、执行跟踪和进度管理能力。该系统通过`PlanNotebook`类实现，支持复杂任务的分解、状态跟踪、历史记录和恢复功能。计划系统采用分层架构，包含计划模型、存储后端和执行接口三个核心组件，为智能体提供从任务创建到完成的全生命周期管理。

**Section sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L1-L50)
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L1-L20)

## 核心组件
计划系统由三个核心组件构成：`PlanNotebook`作为主控制器，`Plan`和`SubTask`作为数据模型，以及`PlanStorageBase`作为存储抽象层。这些组件协同工作，实现任务规划的完整功能。

```mermaid
classDiagram
class PlanNotebook {
+max_tasks int
+plan_to_hint Callable
+storage PlanStorageBase
+current_plan Plan
+_plan_change_hooks dict
+__init__(max_subtasks, plan_to_hint, storage)
+create_plan(name, description, expected_outcome, subtasks)
+revise_current_plan(subtask_idx, action, subtask)
+update_subtask_state(subtask_idx, state)
+finish_subtask(subtask_idx, subtask_outcome)
+view_subtasks(subtask_idx)
+finish_plan(state, outcome)
+view_historical_plans()
+recover_historical_plan(plan_id)
+list_tools()
+get_current_hint()
+register_plan_change_hook(hook_name, hook)
+remove_plan_change_hook(hook_name)
}
class Plan {
+id str
+name str
+description str
+expected_outcome str
+subtasks list[SubTask]
+created_at str
+state str
+finished_at str
+outcome str
+refresh_plan_state() str
+finish(state, outcome)
+to_markdown(detailed)
}
class SubTask {
+name str
+description str
+expected_outcome str
+outcome str
+state str
+created_at str
+finished_at str
+finish(outcome)
+to_oneline_markdown()
+to_markdown(detailed)
}
class PlanStorageBase {
<<abstract>>
+add_plan(plan)
+delete_plan(plan_id)
+get_plans()
+get_plan(plan_id)
}
class InMemoryPlanStorage {
+plans OrderedDict
+__init__()
+add_plan(plan, override)
+delete_plan(plan_id)
+get_plans()
+get_plan(plan_id)
}
PlanNotebook --> Plan : "管理"
PlanNotebook --> PlanStorageBase : "使用"
Plan --> SubTask : "包含"
PlanStorageBase <|-- InMemoryPlanStorage : "实现"
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L172-L897)
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L11-L201)
- [storage_base.py](file://src/agentscope/plan/_storage_base.py#L9-L27)
- [in_memory_storage.py](file://src/agentscope/plan/_in_memory_storage.py#L9-L71)

**Section sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L172-L897)
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L11-L201)

## 任务规划接口
`PlanNotebook`类提供了完整的任务规划接口，包括计划创建、修改、执行和完成等操作。这些接口通过工具函数的形式提供给智能体使用，支持复杂任务的分步执行。

### 计划创建与初始化
通过`create_plan`方法创建新计划，需要提供计划名称、描述、预期结果和子任务列表。如果已有计划存在，新计划将替换当前计划。

```mermaid
sequenceDiagram
participant Agent as "智能体"
participant PlanNotebook as "PlanNotebook"
Agent->>PlanNotebook : create_plan(name, description, expected_outcome, subtasks)
PlanNotebook->>PlanNotebook : 创建Plan实例
PlanNotebook->>PlanNotebook : 设置current_plan
PlanNotebook->>PlanNotebook : 触发计划变更钩子
PlanNotebook-->>Agent : 返回创建成功消息
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L232-L292)

### 计划修改接口
`revise_current_plan`方法支持对当前计划进行修改，包括添加、修改和删除子任务三种操作。该接口确保计划的灵活性，允许在执行过程中根据实际情况调整计划。

```mermaid
flowchart TD
Start([开始修改计划]) --> ValidateInput["验证输入参数"]
ValidateInput --> InputValid{"参数有效?"}
InputValid --> |否| ReturnError["返回错误信息"]
InputValid --> |是| CheckAction["检查操作类型"]
CheckAction --> Add{"添加?"}
CheckAction --> Revise{"修改?"}
CheckAction --> Delete{"删除?"}
Add --> |是| InsertSubtask["在指定位置插入子任务"]
Revise --> |是| UpdateSubtask["更新指定位置的子任务"]
Delete --> |是| RemoveSubtask["删除指定位置的子任务"]
InsertSubtask --> TriggerHook["触发计划变更钩子"]
UpdateSubtask --> TriggerHook
RemoveSubtask --> TriggerHook
TriggerHook --> ReturnSuccess["返回成功消息"]
ReturnError --> End([结束])
ReturnSuccess --> End
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L302-L425)

### 子任务状态管理
`update_subtask_state`和`finish_subtask`方法用于管理子任务的执行状态。系统强制执行顺序约束，确保只有当前子任务完成后才能开始下一个子任务。

```mermaid
sequenceDiagram
participant Agent as "智能体"
participant PlanNotebook as "PlanNotebook"
Agent->>PlanNotebook : update_subtask_state(subtask_idx, "in_progress")
PlanNotebook->>PlanNotebook : 验证当前计划存在
PlanNotebook->>PlanNotebook : 验证子任务索引有效
PlanNotebook->>PlanNotebook : 检查前置子任务是否完成
PlanNotebook->>PlanNotebook : 检查是否有其他子任务正在执行
PlanNotebook->>PlanNotebook : 更新子任务状态
PlanNotebook->>PlanNotebook : 更新计划状态
PlanNotebook->>PlanNotebook : 触发计划变更钩子
PlanNotebook-->>Agent : 返回状态更新成功消息
Agent->>PlanNotebook : finish_subtask(subtask_idx, outcome)
PlanNotebook->>PlanNotebook : 验证当前计划存在
PlanNotebook->>PlanNotebook : 验证子任务索引有效
PlanNotebook->>PlanNotebook : 检查所有前置子任务是否完成
PlanNotebook->>PlanNotebook : 标记子任务为完成状态
PlanNotebook->>PlanNotebook : 自动激活下一个子任务
PlanNotebook->>PlanNotebook : 触发计划变更钩子
PlanNotebook-->>Agent : 返回完成成功消息
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L427-L645)

## 计划存储后端
计划存储系统采用抽象接口设计，支持多种存储实现。当前提供了内存存储作为默认实现，同时为扩展其他存储方式提供了接口。

### 存储接口定义
`PlanStorageBase`是所有计划存储实现的抽象基类，定义了四个核心接口：添加计划、删除计划、获取所有计划和按ID获取计划。

```mermaid
classDiagram
class PlanStorageBase {
<<abstract>>
+add_plan(plan)
+delete_plan(plan_id)
+get_plans()
+get_plan(plan_id)
}
PlanStorageBase <|-- InMemoryPlanStorage : "实现"
class InMemoryPlanStorage {
+plans OrderedDict
+__init__()
+add_plan(plan, override)
+delete_plan(plan_id)
+get_plans()
+get_plan(plan_id)
}
```

**Diagram sources**
- [storage_base.py](file://src/agentscope/plan/_storage_base.py#L9-L27)
- [in_memory_storage.py](file://src/agentscope/plan/_in_memory_storage.py#L9-L71)

### 内存存储实现
`InMemoryPlanStorage`使用有序字典存储计划，支持计划的序列化和反序列化，确保状态的持久化。

```mermaid
flowchart TD
A[添加计划] --> B{计划ID已存在?}
B --> |是| C{允许覆盖?}
C --> |否| D[抛出异常]
C --> |是| E[覆盖现有计划]
B --> |否| E
E --> F[存储到plans字典]
G[获取所有计划] --> H[返回plans字典的值列表]
I[按ID获取计划] --> J{计划ID存在?}
J --> |是| K[返回对应计划]
J --> |否| L[返回None]
M[删除计划] --> N[从plans字典中移除指定ID的计划]
```

**Diagram sources**
- [in_memory_storage.py](file://src/agentscope/plan/_in_memory_storage.py#L26-L71)

**Section sources**
- [storage_base.py](file://src/agentscope/plan/_storage_base.py#L9-L27)
- [in_memory_storage.py](file://src/agentscope/plan/_in_memory_storage.py#L9-L71)

## 计划与智能体联动
计划系统通过提示生成和状态同步机制与智能体紧密集成，确保智能体能够按照计划执行任务。

### 提示生成机制
`DefaultPlanToHint`类根据当前计划状态生成相应的提示信息，指导智能体下一步操作。系统提供了四种状态的提示模板：无计划、计划开始、子任务进行中和所有子任务完成后。

```mermaid
flowchart TD
A[生成提示] --> B{当前计划为空?}
B --> |是| C[返回创建计划提示]
B --> |否| D[统计各状态子任务数量]
D --> E{进行中的子任务数量>0?}
E --> |是| F[返回进行中提示]
E --> |否| G{已完成的子任务数量>0?}
G --> |是| H[返回无进行中提示]
G --> |否| I[返回计划开始提示]
F --> J{所有子任务都已完成或已放弃?}
J --> |是| K[返回计划完成提示]
J --> |否| F
K --> L[格式化提示信息]
L --> M[添加系统提示标签]
M --> N[返回最终提示]
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L16-L168)

### 状态同步接口
`get_current_hint`方法获取当前计划的提示信息，`_trigger_plan_change_hooks`方法在计划变更时触发所有注册的钩子函数。

```mermaid
sequenceDiagram
participant Agent as "智能体"
participant PlanNotebook as "PlanNotebook"
participant Hook as "计划变更钩子"
Agent->>PlanNotebook : get_current_hint()
PlanNotebook->>PlanNotebook : 调用plan_to_hint函数
PlanNotebook->>PlanNotebook : 生成提示内容
PlanNotebook->>PlanNotebook : 包装为Msg对象
PlanNotebook-->>Agent : 返回提示消息
PlanNotebook->>PlanNotebook : _trigger_plan_change_hooks()
PlanNotebook->>Hook : 调用每个注册的钩子函数
loop 所有钩子函数
Hook->>Hook : 执行钩子逻辑
end
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L838-L897)

**Section sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L16-L168)
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L838-L897)

## 多层级计划管理
计划系统支持多层级任务管理，通过子任务列表实现任务的层次化分解和依赖关系管理。

### 依赖关系与优先级
系统通过子任务的顺序隐式定义依赖关系，强制执行顺序约束。只有当前子任务完成后，才能开始下一个子任务。

```mermaid
erDiagram
PLAN ||--o{ SUBTASK : "包含"
PLAN {
string id PK
string name
string description
string expected_outcome
string created_at
string state
string finished_at
string outcome
}
SUBTASK {
string name PK, FK
string description
string expected_outcome
string outcome
string state
string created_at
string finished_at
}
```

**Diagram sources**
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L104-L201)

### 优先级调度算法
系统采用简单的顺序调度算法，按照子任务在列表中的顺序依次执行。通过`refresh_plan_state`方法维护计划的整体状态。

```mermaid
flowchart TD
A[刷新计划状态] --> B{计划状态为完成或已放弃?}
B --> |是| C[不进行状态更新]
B --> |否| D{是否存在进行中的子任务?}
D --> |是| E{计划状态为待办?}
E --> |是| F[更新计划状态为进行中]
E --> |否| G[保持当前状态]
D --> |否| H{计划状态为进行中?}
H --> |是| I[更新计划状态为待办]
H --> |否| J[保持当前状态]
```

**Diagram sources**
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L148-L167)

**Section sources**
- [plan_model.py](file://src/agentscope/plan/_plan_model.py#L104-L201)

## 计划可视化与人工干预
计划系统提供了完整的可视化和人工干预接口，支持外部系统监控和干预计划执行过程。

### 可视化接口
通过`register_plan_change_hook`和`remove_plan_change_hook`方法，外部系统可以注册和移除计划变更钩子，实现计划的实时可视化。

```mermaid
sequenceDiagram
participant Frontend as "前端界面"
participant PlanNotebook as "PlanNotebook"
Frontend->>PlanNotebook : register_plan_change_hook("visualization", hook_func)
PlanNotebook->>PlanNotebook : 将钩子函数添加到_plan_change_hooks
PlanNotebook-->>Frontend : 注册成功
PlanNotebook->>Frontend : 触发钩子函数(PlanNotebook, current_plan)
Frontend->>Frontend : 更新可视化界面
Frontend->>PlanNotebook : remove_plan_change_hook("visualization")
PlanNotebook->>PlanNotebook : 从_plan_change_hooks中移除钩子
PlanNotebook-->>Frontend : 移除成功
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L859-L888)

### 人工干预编程接入
`view_subtasks`和`view_historical_plans`方法提供了计划查看接口，`recover_historical_plan`方法支持从历史计划恢复。

```mermaid
flowchart TD
A[查看子任务] --> B{验证当前计划存在}
B --> |否| C[抛出异常]
B --> |是| D{验证子任务索引}
D --> |无效| E[返回错误信息]
D --> |有效| F[获取子任务详细信息]
F --> G[格式化为Markdown]
G --> H[返回结果]
I[恢复历史计划] --> J{获取指定ID的计划}
J --> |不存在| K[返回错误信息]
J --> |存在| L{当前计划存在?}
L --> |是| M[完成当前计划]
L --> |否| N[直接恢复]
M --> O[存储当前计划到历史]
O --> P[设置当前计划为历史计划]
P --> Q[返回成功消息]
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L647-L812)

**Section sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L859-L888)
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L647-L812)

## 异常恢复策略
计划系统提供了完善的异常恢复机制，确保在意外中断后能够恢复执行状态。

### 状态持久化
通过`state_dict`和`load_state_dict`方法，系统支持计划状态的序列化和反序列化，确保状态的持久化存储。

```mermaid
sequenceDiagram
participant Agent as "智能体"
participant PlanNotebook as "PlanNotebook"
Agent->>PlanNotebook : state_dict()
PlanNotebook->>PlanNotebook : 序列化current_plan
PlanNotebook->>PlanNotebook : 序列化storage状态
PlanNotebook-->>Agent : 返回状态字典
Agent->>PlanNotebook : load_state_dict(state_dict)
PlanNotebook->>PlanNotebook : 反序列化current_plan
PlanNotebook->>PlanNotebook : 反序列化storage状态
PlanNotebook-->>Agent : 状态恢复完成
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L225-L229)

### 计划完成处理
`finish_plan`方法在完成计划时，会将计划存储到历史记录中，并清理当前计划状态。

```mermaid
flowchart TD
A[完成计划] --> B{验证当前计划存在}
B --> |否| C[返回错误信息]
B --> |是| D[更新计划状态]
D --> E[存储到历史记录]
E --> F[清理当前计划]
F --> G[触发计划变更钩子]
G --> H[返回成功消息]
```

**Diagram sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L688-L731)

**Section sources**
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L225-L229)
- [plan_notebook.py](file://src/agentscope/plan/_plan_notebook.py#L688-L731)