---
title: Spring Boot 整合 Activiti 实战：支持前端在线绘制流程图，并通过 JSON 自动部署流程
---

# Spring Boot 整合 Activiti 实战：支持前端在线绘制流程图，并通过 JSON 自动部署流程

> 适用场景：OA、请假、报销、合同审批、采购审批、工单流转、售后工单、业务编排。
>
> 文章目标：从 0 到 1 搭建一个 **Spring Boot + Activiti + MySQL + bpmn-js** 的工作流系统，做到：
>
> 1. 后端完成 Activiti 基础集成。
> 2. 支持流程定义、流程部署、启动流程、待办任务、审批、驳回、历史查询。
> 3. 前端可直接在线绘制 BPMN 流程图。
> 4. 前端通过 **JSON 请求体** 提交流程图数据，后端自动部署流程。
> 5. 在 **不强绑定某个前端框架** 的前提下，给出 Vue/React 都好接入的方案。

---

## 1. 为什么选择 Activiti

Activiti 是一个轻量级、Java 生态友好的 BPMN 2.0 工作流引擎，适合在 Spring Boot 项目中快速落地审批流、任务流和业务流程自动化。它的核心能力包括：

- **流程定义与部署**：将 BPMN 2.0 流程定义部署到引擎。
- **流程实例管理**：启动、挂起、激活、终止流程实例。
- **任务管理**：查询待办、签收、完成、委派、转办。
- **历史数据追踪**：查询流程执行历史、任务历史、变量历史。
- **流程变量**：支持业务参数在整个流程中的流转。
- **网关与条件流转**：支持排他网关、并行网关、包含网关等。
- **监听器与扩展点**：可以在节点进入、离开、任务创建、任务完成时扩展业务逻辑。

如果你的场景本质上是“**一组步骤按照规则流转，并且每个节点需要人或系统来处理**”，那 Activiti 就很适合。

---

## 2. 先说结论：前端画图 + JSON 自动部署，推荐怎么做

很多人第一次做“在线流程设计器”时，会纠结“前端到底传 XML 还是传 JSON”。这里直接给结论：

### 推荐方案：JSON 外壳 + BPMN XML 内容

前端用 `bpmn-js` 在线绘制流程图。绘制完成后导出 BPMN XML，然后将 XML 放进一个 JSON 请求体中提交给后端，例如：

```json
{
  "processKey": "leaveApproval",
  "processName": "请假审批流程",
  "category": "oa",
  "tenantId": "default",
  "bpmnXml": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>..."
}
```

这样做的优点：

1. **前端仍然是传 JSON**，满足接口规范。
2. `bpmn-js` 原生最擅长导出的是 BPMN XML，直接部署最稳。
3. 后端无需做复杂的“自定义节点 JSON -> BPMN XML”转换。
4. 与 Activiti 的部署接口天然匹配，维护成本最低。

### 也支持纯业务 JSON 建模

如果你的前端不是标准 BPMN 设计器，而是你自己封装的一套“节点、连线、审批人、条件表达式”的 JSON 结构，也可以在后端把该 JSON 转成 `BpmnModel` 再生成 XML 并部署。

本文会把两种方式都讲清楚，但实际项目里建议优先采用：

**前端画 BPMN -> 导出 XML -> 放入 JSON 请求体 -> 后端自动部署**。

---

## 3. 技术选型建议

本文示例选型如下：

- **JDK**：17
- **Spring Boot**：2.7.x 或你当前公司稳定版本
- **Activiti**：与 Spring Boot 兼容的稳定版本
- **数据库**：MySQL 8.x
- **前端流程设计器**：bpmn-js
- **构建工具**：Maven

### 版本建议说明

Activiti 在不同分支、不同 Starter、不同仓库之间版本会比较分散，因此生产环境一定要以“**你当前项目的 Spring Boot 版本、JDK 版本、依赖兼容性**”为准，而不是只看某一个博客里的版本号。比较稳妥的做法是：

1. 先固定你的 Spring Boot 版本。
2. 选一个与你项目兼容的 Activiti Starter。
3. 在 PoC 环境把流程部署、任务流转、历史查询、事务一致性全部跑通后再进入生产。

---

## 4. Activiti 的核心表结构和核心服务

Spring Boot 启动后，Activiti 会自动建表（前提是配置允许自动建表）。常见表按用途可分为几类：

### 4.1 流程定义相关

- `ACT_RE_DEPLOYMENT`：部署信息
- `ACT_RE_PROCDEF`：流程定义
- `ACT_GE_BYTEARRAY`：BPMN XML、PNG 等二进制资源

### 4.2 运行时相关

- `ACT_RU_EXECUTION`：执行实例
- `ACT_RU_TASK`：运行中的任务
- `ACT_RU_VARIABLE`：运行中的流程变量
- `ACT_RU_IDENTITYLINK`：任务与用户/组关系

### 4.3 历史相关

- `ACT_HI_PROCINST`：历史流程实例
- `ACT_HI_TASKINST`：历史任务实例
- `ACT_HI_ACTINST`：历史活动实例
- `ACT_HI_VARINST`：历史变量

### 4.4 身份与权限相关

- `ACT_ID_USER`：用户
- `ACT_ID_GROUP`：组
- `ACT_ID_MEMBERSHIP`：用户-组关系

在实际项目中，很多团队不会直接使用 Activiti 的内置用户体系，而是使用自己系统的用户、角色、组织架构，只把“处理人、候选组、变量”传给引擎。

---

## 5. Activiti 中最重要的几个服务

Activiti 的 API 主要围绕一组 Service 展开：

- `RepositoryService`：流程定义管理、部署管理
- `RuntimeService`：启动流程、删除流程实例、设置变量
- `TaskService`：查询待办、签收任务、完成任务
- `HistoryService`：历史流程、历史任务查询
- `ManagementService`：数据库表、作业、引擎管理
- `IdentityService`：用户组管理
- `FormService`：表单管理

后端绝大多数业务场景，其实都能归纳为下面这五件事：

1. 发布流程
2. 启动流程
3. 查询我的待办
4. 审批任务
5. 查询流程历史

---

## 6. 创建 Spring Boot 工程

### 6.1 Maven 依赖

下面给出一个可落地的依赖模板。注意：**版本号请按你的项目兼容性调整**。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.activiti</groupId>
        <artifactId>activiti-spring-boot-starter</artifactId>
        <version>${activiti.version}</version>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 6.2 application.yml

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/activiti_demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&nullCatalogMeansCurrent=true
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai

activiti:
  database-schema-update: true
  db-history-used: true
  history-level: full
  check-process-definitions: false
```

### 6.3 启动类

```java
@SpringBootApplication
public class ActivitiDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(ActivitiDemoApplication.class, args);
    }
}
```

启动成功后，数据库中应该会自动创建一批 `ACT_` 开头的表。

---

## 7. 先理解 BPMN：你以后会一直用到的节点

在实际审批系统里，最常用的 BPMN 元素如下：

### 7.1 StartEvent 开始事件

流程入口。一个流程一般至少要有一个开始事件。

### 7.2 UserTask 用户任务

表示某个节点需要人工处理，例如：

- 部门经理审批
- 财务审批
- 人事审批
- 总经理审批

### 7.3 ExclusiveGateway 排他网关

表示条件分支，只会走其中一条路径。例如：

- 请假天数 <= 3 天，走部门经理审批
- 请假天数 > 3 天，走总监审批

### 7.4 ParallelGateway 并行网关

表示并行处理。例如同时发给法务和财务审批，两个都完成后再往下走。

### 7.5 ServiceTask 服务任务

表示由系统自动执行的节点，例如：

- 自动生成单号
- 调用第三方接口
- 发送短信/邮件/站内信
- 调用 ERP / CRM / OA 系统接口

### 7.6 EndEvent 结束事件

流程结束。

---

## 8. 一个最小可运行的请假流程 BPMN 示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns:activiti="http://activiti.org/bpmn"
             targetNamespace="http://www.example.com/processdef">

    <process id="leaveApproval" name="请假审批流程" isExecutable="true">
        <startEvent id="startEvent" name="开始"/>

        <userTask id="deptLeaderApprove" name="部门经理审批"
                  activiti:assignee="${deptLeader}"/>

        <exclusiveGateway id="daysGateway" name="请假天数判断"/>

        <userTask id="hrApprove" name="HR审批"
                  activiti:assignee="${hr}"/>

        <userTask id="directorApprove" name="总监审批"
                  activiti:assignee="${director}"/>

        <endEvent id="endEvent" name="结束"/>

        <sequenceFlow id="flow1" sourceRef="startEvent" targetRef="deptLeaderApprove"/>
        <sequenceFlow id="flow2" sourceRef="deptLeaderApprove" targetRef="daysGateway"/>
        <sequenceFlow id="flow3" sourceRef="daysGateway" targetRef="hrApprove">
            <conditionExpression xsi:type="tFormalExpression"><![CDATA[${days <= 3}]]></conditionExpression>
        </sequenceFlow>
        <sequenceFlow id="flow4" sourceRef="daysGateway" targetRef="directorApprove">
            <conditionExpression xsi:type="tFormalExpression"><![CDATA[${days > 3}]]></conditionExpression>
        </sequenceFlow>
        <sequenceFlow id="flow5" sourceRef="hrApprove" targetRef="endEvent"/>
        <sequenceFlow id="flow6" sourceRef="directorApprove" targetRef="endEvent"/>
    </process>
</definitions>
```

这个流程已经覆盖了审批流里最常见的元素：开始、人工审批、条件判断、结束。

---

## 9. 后端实现：流程部署

### 9.1 定义请求对象

```java
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class ProcessDeployRequest {

    @NotBlank(message = "流程Key不能为空")
    private String processKey;

    @NotBlank(message = "流程名称不能为空")
    private String processName;

    private String category;

    private String tenantId;

    /**
     * 前端通过 bpmn-js 导出的 BPMN XML
     */
    @NotBlank(message = "bpmnXml不能为空")
    private String bpmnXml;
}
```

### 9.2 部署服务

```java
import lombok.RequiredArgsConstructor;
import org.activiti.engine.RepositoryService;
import org.activiti.engine.repository.Deployment;
import org.springframework.stereotype.Service;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;

@Service
@RequiredArgsConstructor
public class ProcessDeployService {

    private final RepositoryService repositoryService;

    public String deploy(ProcessDeployRequest request) {
        String resourceName = request.getProcessKey() + ".bpmn20.xml";

        Deployment deployment = repositoryService.createDeployment()
                .name(request.getProcessName())
                .category(request.getCategory())
                .tenantId(request.getTenantId())
                .addInputStream(
                        resourceName,
                        new ByteArrayInputStream(request.getBpmnXml().getBytes(StandardCharsets.UTF_8))
                )
                .deploy();

        return deployment.getId();
    }
}
```

### 9.3 控制器

```java
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/process")
@RequiredArgsConstructor
public class ProcessController {

    private final ProcessDeployService processDeployService;

    @PostMapping("/deploy")
    public Map<String, Object> deploy(@RequestBody @Valid ProcessDeployRequest request) {
        String deploymentId = processDeployService.deploy(request);
        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("deploymentId", deploymentId);
        return result;
    }
}
```

至此，你已经完成了“**前端传 JSON，后端自动部署流程**”这件事。

这里的关键点在于：

- 请求体是 JSON。
- JSON 中包含 `bpmnXml` 字段。
- 后端将 XML 作为部署资源提交给 Activiti。

这就是最稳、最易维护、最符合工程实践的实现方式。

---

## 10. 后端实现：启动流程实例

### 10.1 请求对象

```java
import lombok.Data;

import java.util.Map;

@Data
public class ProcessStartRequest {
    private String processDefinitionKey;
    private String businessKey;
    private Map<String, Object> variables;
}
```

### 10.2 启动流程服务

```java
import lombok.RequiredArgsConstructor;
import org.activiti.engine.RuntimeService;
import org.activiti.engine.runtime.ProcessInstance;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ProcessRuntimeAppService {

    private final RuntimeService runtimeService;

    public String start(ProcessStartRequest request) {
        ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(
                request.getProcessDefinitionKey(),
                request.getBusinessKey(),
                request.getVariables()
        );
        return processInstance.getProcessInstanceId();
    }
}
```

### 10.3 控制器

```java
@RestController
@RequestMapping("/api/runtime")
@RequiredArgsConstructor
public class RuntimeController {

    private final ProcessRuntimeAppService processRuntimeAppService;

    @PostMapping("/start")
    public Map<String, Object> start(@RequestBody ProcessStartRequest request) {
        String processInstanceId = processRuntimeAppService.start(request);
        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        result.put("processInstanceId", processInstanceId);
        return result;
    }
}
```

### 10.4 启动流程示例

```json
{
  "processDefinitionKey": "leaveApproval",
  "businessKey": "LEAVE_20260312001",
  "variables": {
    "days": 5,
    "deptLeader": "zhangsan",
    "hr": "lisi",
    "director": "wangwu",
    "applicant": "hsl"
  }
}
```

这里的 `variables` 就是流程变量，它决定了：

- 当前流程走哪条线
- 节点由谁审批
- 历史记录中保留哪些业务信息

---

## 11. 后端实现：待办任务、签收、审批完成

### 11.1 查询我的待办任务

```java
import lombok.RequiredArgsConstructor;
import org.activiti.engine.TaskService;
import org.activiti.task.api.Task;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class TaskAppService {

    private final TaskService taskService;

    public List<TaskDto> myTodoTasks(String assignee) {
        List<Task> tasks = taskService.createTaskQuery()
                .taskAssignee(assignee)
                .orderByTaskCreateTime()
                .desc()
                .list();

        return tasks.stream().map(task -> {
            TaskDto dto = new TaskDto();
            dto.setTaskId(task.getId());
            dto.setTaskName(task.getName());
            dto.setProcessInstanceId(task.getProcessInstanceId());
            dto.setExecutionId(task.getExecutionId());
            dto.setCreateTime(task.getCreateTime());
            return dto;
        }).collect(Collectors.toList());
    }
}
```

```java
import lombok.Data;

import java.util.Date;

@Data
public class TaskDto {
    private String taskId;
    private String taskName;
    private String processInstanceId;
    private String executionId;
    private Date createTime;
}
```

### 11.2 完成任务

```java
import lombok.Data;

import java.util.Map;

@Data
public class CompleteTaskRequest {
    private String taskId;
    private Map<String, Object> variables;
}
```

```java
@Service
@RequiredArgsConstructor
public class TaskCompleteService {

    private final TaskService taskService;

    public void complete(CompleteTaskRequest request) {
        taskService.complete(request.getTaskId(), request.getVariables());
    }
}
```

### 11.3 控制器

```java
@RestController
@RequestMapping("/api/task")
@RequiredArgsConstructor
public class TaskController {

    private final TaskAppService taskAppService;
    private final TaskCompleteService taskCompleteService;

    @GetMapping("/todo")
    public List<TaskDto> todo(@RequestParam String assignee) {
        return taskAppService.myTodoTasks(assignee);
    }

    @PostMapping("/complete")
    public Map<String, Object> complete(@RequestBody CompleteTaskRequest request) {
        taskCompleteService.complete(request);
        Map<String, Object> result = new HashMap<>();
        result.put("success", true);
        return result;
    }
}
```

审批通过时你可以传：

```json
{
  "taskId": "25005",
  "variables": {
    "approved": true,
    "comment": "同意"
  }
}
```

审批驳回时你也可以传：

```json
{
  "taskId": "25005",
  "variables": {
    "approved": false,
    "comment": "资料不完整，请补充"
  }
}
```

如果你在 BPMN 中已经把驳回逻辑画成条件流转，那么只要变量正确，流程就会自动走对应分支。

---

## 12. 后端实现：流程历史查询

```java
import lombok.RequiredArgsConstructor;
import org.activiti.engine.HistoryService;
import org.activiti.engine.history.HistoricTaskInstance;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class ProcessHistoryService {

    private final HistoryService historyService;

    public List<HistoricTaskDto> queryHistoricTasks(String processInstanceId) {
        List<HistoricTaskInstance> list = historyService.createHistoricTaskInstanceQuery()
                .processInstanceId(processInstanceId)
                .orderByHistoricTaskInstanceStartTime()
                .asc()
                .list();

        return list.stream().map(item -> {
            HistoricTaskDto dto = new HistoricTaskDto();
            dto.setTaskId(item.getId());
            dto.setTaskName(item.getName());
            dto.setAssignee(item.getAssignee());
            dto.setStartTime(item.getStartTime());
            dto.setEndTime(item.getEndTime());
            dto.setDurationInMillis(item.getDurationInMillis());
            return dto;
        }).collect(Collectors.toList());
    }
}
```

```java
import lombok.Data;

import java.util.Date;

@Data
public class HistoricTaskDto {
    private String taskId;
    private String taskName;
    private String assignee;
    private Date startTime;
    private Date endTime;
    private Long durationInMillis;
}
```

---

## 13. 前端在线流程设计：为什么推荐 bpmn-js

`bpmn-js` 是一个成熟的 BPMN 2.0 渲染和建模工具库，适合在浏览器中嵌入流程设计器。它可以：

- 加载现有 BPMN XML
- 在线编辑节点和连线
- 导出 BPMN XML
- 扩展属性面板
- 支持自定义 moddle 扩展

这正好符合“前端直接画流程图”的要求。

---

## 14. 前端接入 bpmn-js 的基本思路

### 14.1 安装依赖

```bash
npm install bpmn-js
```

如果还需要属性面板：

```bash
npm install bpmn-js-properties-panel @bpmn-io/properties-panel
```

### 14.2 一个最小可运行示例

```html
<div id="canvas" style="width: 100%; height: 600px; border: 1px solid #ddd;"></div>
<button id="saveBtn">保存并部署</button>

<script type="module">
  import BpmnModeler from 'bpmn-js/lib/Modeler';

  const modeler = new BpmnModeler({
    container: '#canvas'
  });

  const defaultXml = `<?xml version="1.0" encoding="UTF-8"?>
  <bpmn:definitions xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                    xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                    xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                    xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                    xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                    xmlns:activiti="http://activiti.org/bpmn"
                    targetNamespace="http://bpmn.io/schema/bpmn">
    <bpmn:process id="leaveApproval" isExecutable="true">
      <bpmn:startEvent id="StartEvent_1" />
    </bpmn:process>
    <bpmndi:BPMNDiagram id="BPMNDiagram_1">
      <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="leaveApproval">
        <bpmndi:BPMNShape id="StartEvent_1_di" bpmnElement="StartEvent_1">
          <dc:Bounds x="173" y="102" width="36" height="36" />
        </bpmndi:BPMNShape>
      </bpmndi:BPMNPlane>
    </bpmndi:BPMNDiagram>
  </bpmn:definitions>`;

  await modeler.importXML(defaultXml);

  document.getElementById('saveBtn').addEventListener('click', async () => {
    const { xml } = await modeler.saveXML({ format: true });

    const body = {
      processKey: 'leaveApproval',
      processName: '请假审批流程',
      category: 'oa',
      tenantId: 'default',
      bpmnXml: xml
    };

    await fetch('/api/process/deploy', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(body)
    });

    alert('流程部署成功');
  });
</script>
```

这个例子已经实现了完整闭环：

- 浏览器中画图
- 导出 BPMN XML
- 封装为 JSON 请求体
- 提交给后端
- 后端自动部署到 Activiti

---

## 15. 如何在前端设置 Activiti 扩展属性

画流程图只是第一步，真正落地项目时你还要给节点配置业务属性，例如：

- 审批人表达式：`${deptLeader}`
- 候选组：`financeGroup`
- 条件表达式：`${amount > 10000}`
- 服务任务 Bean：`${sendMailDelegate}`
- 节点表单 Key
- 超时提醒配置

这些都可以通过：

1. 属性面板直接写入 BPMN XML 扩展属性。
2. 自定义侧边栏，把节点属性单独保存在业务表里。
3. 两者混合：关键流转属性进 XML，纯业务展示属性进业务表。

### 推荐实践

- **影响流程流转的属性**：放进 BPMN XML，例如 assignee、candidateGroup、conditionExpression。
- **仅用于业务展示的属性**：放业务库，例如按钮权限、表单元数据、前端渲染配置。

这样不会把 BPMN 搞得过重，也方便维护。

---

## 16. 如果前端真的只传“纯 JSON 节点数据”，后端怎么自动部署

有些项目不想让前端关心 XML，只想传这种 JSON：

```json
{
  "processKey": "expenseApproval",
  "processName": "报销审批流程",
  "nodes": [
    {"id": "start", "type": "startEvent", "name": "开始"},
    {"id": "apply", "type": "userTask", "name": "提交申请", "assignee": "${applicant}"},
    {"id": "manager", "type": "userTask", "name": "经理审批", "assignee": "${manager}"},
    {"id": "end", "type": "endEvent", "name": "结束"}
  ],
  "flows": [
    {"id": "f1", "source": "start", "target": "apply"},
    {"id": "f2", "source": "apply", "target": "manager"},
    {"id": "f3", "source": "manager", "target": "end"}
  ]
}
```

这种情况下，后端需要做两步：

1. 将业务 JSON 转成 `BpmnModel`。
2. 将 `BpmnModel` 转成 BPMN XML 并部署。

### 16.1 JSON 转 BpmnModel 的大致代码骨架

```java
import org.activiti.bpmn.model.*;
import org.activiti.bpmn.converter.BpmnXMLConverter;

public class JsonToBpmnConverter {

    public String toXml(CustomProcessJson json) {
        BpmnModel model = new BpmnModel();
        Process process = new Process();
        process.setId(json.getProcessKey());
        process.setName(json.getProcessName());
        model.addProcess(process);

        for (CustomNode node : json.getNodes()) {
            FlowElement element = buildFlowElement(node);
            process.addFlowElement(element);
        }

        for (CustomFlow flow : json.getFlows()) {
            SequenceFlow sequenceFlow = new SequenceFlow();
            sequenceFlow.setId(flow.getId());
            sequenceFlow.setSourceRef(flow.getSource());
            sequenceFlow.setTargetRef(flow.getTarget());
            sequenceFlow.setConditionExpression(flow.getConditionExpression());
            process.addFlowElement(sequenceFlow);
        }

        byte[] bytes = new BpmnXMLConverter().convertToXML(model);
        return new String(bytes, java.nio.charset.StandardCharsets.UTF_8);
    }

    private FlowElement buildFlowElement(CustomNode node) {
        switch (node.getType()) {
            case "startEvent":
                StartEvent startEvent = new StartEvent();
                startEvent.setId(node.getId());
                startEvent.setName(node.getName());
                return startEvent;
            case "userTask":
                UserTask userTask = new UserTask();
                userTask.setId(node.getId());
                userTask.setName(node.getName());
                userTask.setAssignee(node.getAssignee());
                return userTask;
            case "exclusiveGateway":
                ExclusiveGateway gateway = new ExclusiveGateway();
                gateway.setId(node.getId());
                gateway.setName(node.getName());
                return gateway;
            case "endEvent":
                EndEvent endEvent = new EndEvent();
                endEvent.setId(node.getId());
                endEvent.setName(node.getName());
                return endEvent;
            default:
                throw new IllegalArgumentException("暂不支持的节点类型: " + node.getType());
        }
    }
}
```

### 16.2 再调用部署逻辑

```java
public String deployByJson(CustomProcessJson json) {
    String xml = jsonToBpmnConverter.toXml(json);

    ProcessDeployRequest request = new ProcessDeployRequest();
    request.setProcessKey(json.getProcessKey());
    request.setProcessName(json.getProcessName());
    request.setCategory(json.getCategory());
    request.setTenantId(json.getTenantId());
    request.setBpmnXml(xml);

    return processDeployService.deploy(request);
}
```

### 16.3 什么时候需要这种方式

仅在以下场景推荐纯 JSON 建模：

- 你的前端设计器不是 BPMN 标准设计器。
- 你希望界面更“业务化”，隐藏 BPMN 概念。
- 你只支持有限节点类型，不需要完整 BPMN 能力。

否则，还是推荐 **bpmn-js 出 XML，JSON 包一层后直接部署**。

---

## 17. 任务办理时最常用的功能点

实际项目里，Activiti 常见需求通常不止“完成任务”这么简单，还包括：

### 17.1 签收任务

适用于候选人任务。

```java
taskService.claim(taskId, userId);
```

### 17.2 任务委派

```java
taskService.delegateTask(taskId, userId);
```

### 17.3 设置任务变量

```java
taskService.setVariable(taskId, "approved", true);
```

### 17.4 流程变量

```java
runtimeService.setVariable(processInstanceId, "amount", 12000);
```

### 17.5 删除流程实例

```java
runtimeService.deleteProcessInstance(processInstanceId, "手动作废");
```

### 17.6 查询历史流程实例

```java
historyService.createHistoricProcessInstanceQuery()
        .processInstanceId(processInstanceId)
        .singleResult();
```

---

## 18. 如何做“审批意见、审批记录、流程轨迹”

这是大多数 OA 项目都会问到的点。

### 18.1 审批意见

最常见做法不是完全依赖引擎表，而是单独建业务表，比如：

```sql
CREATE TABLE biz_approval_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    process_instance_id VARCHAR(64) NOT NULL,
    task_id VARCHAR(64) NOT NULL,
    node_key VARCHAR(64) NOT NULL,
    node_name VARCHAR(128) NOT NULL,
    assignee VARCHAR(64),
    action VARCHAR(32) NOT NULL,
    comment_text VARCHAR(2000),
    created_time DATETIME NOT NULL
);
```

这样你可以很灵活地展示：

- 同意
- 驳回
- 转办
- 加签
- 撤回
- 重新提交

### 18.2 流程轨迹

通常有两种做法：

1. 根据历史节点查询 `HistoricActivityInstance`，高亮已走过的节点。
2. 前端加载 BPMN XML 后，根据后端返回的“已完成节点 ID、当前活动节点 ID”做高亮。

### 18.3 审批时间线

可以把以下数据组合成一个时间线接口：

- 流程发起时间
- 每个节点的签收时间
- 每个节点的完成时间
- 审批意见
- 当前停留节点

---

## 19. 常见高级需求的设计建议

### 19.1 驳回到任意节点

标准 BPMN 更多强调“按建模好的路径流转”。如果你要实现“驳回到任意历史节点”，通常需要：

- 在流程模型中预留退回线，或
- 使用更复杂的运行时跳转策略，或
- 通过业务层封装撤回/重提逻辑

这个需求往往不是简单一行代码能优雅解决，建议在建模阶段就设计清楚。

### 19.2 加签 / 会签

可以通过：

- 并行网关
- 多实例用户任务（会签）
- 动态生成子任务

来实现。

### 19.3 抄送

抄送通常不应该直接建成“阻塞节点”，而更适合作为业务层的附加能力：

- 完成任务时写一条抄送记录
- 发消息给相关用户
- 在“我的抄送”列表中展示

### 19.4 流程版本管理

新流程部署后，Activiti 会生成新的流程定义版本。一般建议：

- 老流程实例继续按老版本跑完
- 新发起的流程走新版本
- 不要轻易去强制迁移正在运行的实例

---

## 20. 生产环境落地建议

### 20.1 流程定义与业务表分离

推荐把“流程定义管理”和“业务单据”分离：

- 流程定义：由流程平台维护
- 业务单据：由业务系统维护
- 两者通过 `businessKey` 关联

例如：

- 请假单表：`oa_leave`
- 流程实例：`businessKey = LEAVE_20260312001`

### 20.2 用户身份不要强耦合内置用户表

企业项目中通常已有：

- 用户表
- 组织架构表
- 角色表
- 岗位表

这时更推荐把审批人以变量、候选组、表达式的方式传给引擎，而不是重复维护一套 Activiti 用户体系。

### 20.3 流程变量要分层

建议将变量分成三类：

1. **流转变量**：决定走向，例如 `days`、`amount`、`approved`
2. **处理人变量**：例如 `deptLeader`、`hr`、`director`
3. **业务展示变量**：例如申请标题、单据编号、申请人姓名

不要把整个业务对象无脑全塞进流程变量，否则历史表会越来越重。

### 20.4 幂等与重复提交

发起流程、完成任务接口都要考虑重复点击：

- 前端按钮防重复
- 后端幂等校验
- 用业务主键或版本号控制并发

### 20.5 异常处理与事务一致性

完成审批时，往往不只调用 `taskService.complete()`，还会：

- 记录审批意见
- 更新业务单状态
- 发消息通知
- 写审计日志

这些操作建议放在同一个事务边界内，避免“任务已完成但业务状态没更新”这种问题。

---

## 21. 一个完整的接口设计建议

### 流程设计相关

- `POST /api/process/deploy`：部署流程（JSON 内含 BPMN XML）
- `POST /api/process/deploy-json`：部署流程（纯业务 JSON 自动转 BPMN）
- `GET /api/process/definitions`：查询流程定义列表
- `GET /api/process/definition/{id}`：查询流程定义详情
- `GET /api/process/xml/{definitionId}`：获取 BPMN XML

### 流程运行相关

- `POST /api/runtime/start`：启动流程
- `POST /api/runtime/cancel`：作废流程
- `GET /api/runtime/detail/{processInstanceId}`：流程实例详情

### 任务相关

- `GET /api/task/todo`：我的待办
- `GET /api/task/done`：我的已办
- `POST /api/task/claim`：签收任务
- `POST /api/task/complete`：完成任务
- `POST /api/task/delegate`：委派任务

### 历史与轨迹相关

- `GET /api/history/process/{processInstanceId}`：流程历史
- `GET /api/history/activities/{processInstanceId}`：节点轨迹
- `GET /api/history/comments/{processInstanceId}`：审批意见

---

## 22. 常见问题排查

### 22.1 启动时报找不到表

通常是：

- 数据源配置错误
- 数据库权限不足
- `database-schema-update` 未开启

### 22.2 部署时报 XML 解析异常

通常是：

- XML 格式不合法
- BPMN 命名空间不完整
- 节点引用关系错误
- 表达式写错

建议把前端导出的 XML 先保存下来，单独用校验工具或单元测试跑一遍。

### 22.3 启动流程时报找不到定义

通常是：

- `processDefinitionKey` 不匹配
- 流程没有成功部署
- 部署的是旧版本，查询条件写错

### 22.4 任务查不到

通常是：

- assignee 没传对
- 候选人任务未签收
- 节点还没走到目标用户
- 你查的是历史任务，不是运行时任务

### 22.5 条件流转不生效

通常是：

- 流程变量没传
- 变量类型不对
- 表达式语法错误
- 网关配置错误

---

## 23. 推荐的项目分层

为了后续维护，建议按下面方式分层：

```text
com.example.workflow
├── controller
├── service
│   ├── deploy
│   ├── runtime
│   ├── task
│   └── history
├── domain
│   ├── dto
│   ├── entity
│   └── vo
├── converter
├── config
└── infrastructure
```

如果你的系统已经很大，建议再加一层 `application` / `domain` / `infrastructure`，把“工作流平台能力”和“业务审批逻辑”分开。

---

## 24. 一套更贴近真实项目的落地路径

如果你要把这套方案真正用到 OA、ERP、CRM 里，我建议按这个顺序做：

### 第一步：打通最小闭环

- 能在线画图
- 能部署流程
- 能启动流程
- 能查询待办
- 能完成审批

### 第二步：补齐业务关联

- businessKey 绑定业务单
- 审批意见表
- 消息通知
- 流程历史

### 第三步：补齐设计器能力

- 节点属性面板
- 审批人选择器
- 条件表达式编辑器
- 流程版本管理

### 第四步：补齐平台能力

- 多租户
- 权限控制
- 流程发布审核
- 流程回滚
- 监控与审计

---

## 25. 本文最终推荐方案总结

如果你的目标是：

> “Spring Boot 整合 Activiti，并且前端可以直接画流程图，传入 JSON 就能自动部署流程。”

那最推荐的落地方案是：

### 方案 A：推荐，工程上最稳

- 前端：`bpmn-js` 在线绘制 BPMN
- 导出：得到 BPMN XML
- 提交：封装到 JSON 请求体中的 `bpmnXml` 字段
- 后端：调用 `RepositoryService#createDeployment()` 自动部署

也就是：

**前端画图 -> XML -> JSON 包装 -> 后端部署**

### 方案 B：适合自定义设计器

- 前端：传纯业务 JSON（节点 + 连线 + 条件 + 审批人）
- 后端：JSON 转 `BpmnModel`
- 转换：`BpmnXMLConverter` 生成 BPMN XML
- 部署：Activiti 自动发布

也就是：

**前端纯 JSON -> 后端转 BPMN -> 部署**

### 为什么我更推荐方案 A

因为它：

- 改造成本更低
- 与 bpmn-js 原生能力最一致
- 与 Activiti 部署接口最贴近
- 线上稳定性最好
- 后续排查问题最方便

---

## 26. 可直接复用的部署请求示例

```json
{
  "processKey": "expenseApproval",
  "processName": "报销审批流程",
  "category": "finance",
  "tenantId": "default",
  "bpmnXml": "<?xml version=\"1.0\" encoding=\"UTF-8\"?><definitions ...>...</definitions>"
}
```

只要你的前端能把流程图导出为 XML，这个接口就能直接工作。

---

## 27. 后续可以继续扩展的能力

在本文基础上，你还可以继续做：

- 流程设计器属性面板
- 节点审批人规则配置
- 动态表单
- 流程版本比对
- 流程图节点高亮
- 审批时限和超时提醒
- 撤回、退回、转办、加签
- 钉钉 / 企业微信 / 邮件通知
- 多租户流程平台

---

## 28. 参考实现建议

如果你准备正式落地，建议至少补上这些代码与规范：

- 全局异常处理
- 参数校验
- 单元测试与流程测试
- 数据权限控制
- 流程设计权限控制
- BPMN XML 安全校验
- 审批日志与审计日志
- 业务表与流程表的一致性设计

---

## 29. 参考资料

本文实现思路主要基于以下能力模型：

- Activiti 的流程引擎、部署、运行时、任务与历史服务
- bpmn-js 的浏览器建模、导入导出 XML、可扩展建模能力

如果你写 Hexo 文章，建议在文末增加你项目实际使用的版本号、你们公司的审批场景截图、接口出参样例、流程图效果图，这样文章会更完整。

---

## 30. 小结

到这里，你已经有了一套完整方案：

- 用 Spring Boot 整合 Activiti
- 支持前端使用 bpmn-js 在线画流程图
- 支持 JSON 请求体自动部署流程
- 支持启动流程、查询待办、完成审批、查询历史
- 也预留了“纯 JSON -> BPMN -> 自动部署”的扩展路线

真正做项目时，建议你优先把“**部署、启动、待办、审批、历史**”这五个核心能力先打通，然后再逐步补齐：

- 节点属性配置
- 审批意见
- 流程轨迹
- 多版本管理
- 驳回、加签、转办

这样你落地起来会非常稳。
