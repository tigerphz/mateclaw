# MateClaw Workspace Memory 用户隔离改造方案

## 1. 改造目标

将 Agent 的 workspace memory（Markdown 上下文记忆文件）从 **Agent 级别共享** 改造为 **全局共享 + 用户私有混合** 模式。

**核心规则（白名单）**：
- **全局共享**：`AGENTS.md`、`SOUL.md`、`MEMORY.md`（Agent 行为配置、人格、团队经验）
- **用户私有**：其余所有文件（`PROFILE.md`、`memory/YYYY-MM-DD.md`、`structured/*.md`、其他自定义 `.md`）

> 原则：只有 Agent 本身的行为配置、人格、团队级经验是共享的；任何跟"具体某个用户"相关的记忆都自动隔离。

## 2. 数据库变更

### 2.1 新增列

```sql
-- mate_workspace_file 增加 user_id 列
ALTER TABLE mate_workspace_file ADD COLUMN user_id BIGINT NULL COMMENT 'NULL=全局共享, 非NULL=用户私有';

-- 联合索引优化查询
CREATE INDEX idx_agent_user_file ON mate_workspace_file(agent_id, user_id, filename);
```

### 2.2 数据迁移

```sql
-- 现有数据全部标记为全局共享（向后兼容）
UPDATE mate_workspace_file SET user_id = NULL;
```

### 2.3 表结构

```sql
CREATE TABLE IF NOT EXISTS mate_workspace_file (
    id              BIGINT       NOT NULL PRIMARY KEY,
    agent_id        BIGINT       NOT NULL,
    user_id         BIGINT       NULL,           -- 新增
    filename        VARCHAR(256) NOT NULL,
    content         CLOB,
    file_size       BIGINT       NOT NULL DEFAULT 0,
    enabled         BOOLEAN      NOT NULL DEFAULT FALSE,
    sort_order      INT          NOT NULL DEFAULT 0,
    create_time     DATETIME     NOT NULL,
    update_time     DATETIME     NOT NULL,
    deleted         INT          NOT NULL DEFAULT 0
);
```

## 3. 实体类改造

### 3.1 WorkspaceFileEntity

**文件**：`mateclaw-server/src/main/java/vip/mate/workspace/document/model/WorkspaceFileEntity.java`

新增 `userId` 字段：

```java
@Data
@TableName("mate_workspace_file")
public class WorkspaceFileEntity {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    /** 关联的 Agent ID */
    private Long agentId;

    /** 所属用户 ID（NULL = 全局共享，非NULL = 用户私有） */
    private Long userId;  // 新增

    /** 文件名（如 AGENTS.md、SOUL.md） */
    private String filename;

    /** Markdown 内容 */
    @TableField(updateStrategy = FieldStrategy.ALWAYS)
    private String content;

    /** 文件大小（字节） */
    private Long fileSize;

    /** 是否启用为系统提示词 */
    private Boolean enabled;

    /** 排序顺序（越小越靠前） */
    private Integer sortOrder;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    private Integer deleted;
}
```

## 4. Service 层核心改造

### 4.1 WorkspaceFileService

**文件**：`mateclaw-server/src/main/java/vip/mate/workspace/document/WorkspaceFileService.java`

#### 4.1.1 核心判断方法（白名单）

```java
/**
 * 判断文件是否为全局共享类型（白名单）
 * 只有 AGENTS.md、SOUL.md、MEMORY.md 是全局的，其余全部为用户私有
 */
private boolean isGlobalFile(String filename) {
    if (filename == null) return false;
    return "AGENTS.md".equals(filename)
            || "SOUL.md".equals(filename)
            || "MEMORY.md".equals(filename);
}
```

#### 4.1.2 查询方法改造

```java
/**
 * 列出 Agent 的所有工作区文件（全局 + 当前用户私有）
 */
public List<WorkspaceFileEntity> listFiles(Long agentId, Long userId) {
    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId);
    if (userId != null) {
        wrapper.and(w -> w
                .isNull(WorkspaceFileEntity::getUserId)
                .or()
                .eq(WorkspaceFileEntity::getUserId, userId));
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }
    wrapper.orderByAsc(WorkspaceFileEntity::getSortOrder)
           .orderByAsc(WorkspaceFileEntity::getFilename);
    List<WorkspaceFileEntity> files = fileMapper.selectList(wrapper);
    files.forEach(f -> f.setContent(null));
    return files;
}

// 旧签名兼容
public List<WorkspaceFileEntity> listFiles(Long agentId) {
    return listFiles(agentId, null);
}
```

#### 4.1.3 读取单个文件改造

```java
/**
 * 读取单个文件（含内容）
 * 优先查找用户私有文件，不存在则回退到全局文件
 */
public WorkspaceFileEntity getFile(Long agentId, String filename, Long userId) {
    boolean isUserFile = !isGlobalFile(filename);
    if (isUserFile && userId != null) {
        WorkspaceFileEntity userFile = fileMapper.selectOne(
                new LambdaQueryWrapper<WorkspaceFileEntity>()
                        .eq(WorkspaceFileEntity::getAgentId, agentId)
                        .eq(WorkspaceFileEntity::getUserId, userId)
                        .eq(WorkspaceFileEntity::getFilename, filename));
        if (userFile != null) {
            return userFile;
        }
    }
    // 回退到全局文件
    return fileMapper.selectOne(
            new LambdaQueryWrapper<WorkspaceFileEntity>()
                    .eq(WorkspaceFileEntity::getAgentId, agentId)
                    .isNull(WorkspaceFileEntity::getUserId)
                    .eq(WorkspaceFileEntity::getFilename, filename));
}

// 旧签名兼容
public WorkspaceFileEntity getFile(Long agentId, String filename) {
    return getFile(agentId, filename, null);
}
```

#### 4.1.4 保存文件改造

```java
/**
 * 创建或更新文件（支持用户隔离）
 */
@Transactional
public WorkspaceFileEntity saveFile(Long agentId, String filename, String content, Long userId) {
    boolean isUserFile = !isGlobalFile(filename);
    Long targetUserId = (isUserFile && userId != null) ? userId : null;

    WorkspaceFileEntity existing = null;
    if (targetUserId != null) {
        existing = fileMapper.selectOne(
                new LambdaQueryWrapper<WorkspaceFileEntity>()
                        .eq(WorkspaceFileEntity::getAgentId, agentId)
                        .eq(WorkspaceFileEntity::getUserId, targetUserId)
                        .eq(WorkspaceFileEntity::getFilename, filename));
    } else {
        existing = fileMapper.selectOne(
                new LambdaQueryWrapper<WorkspaceFileEntity>()
                        .eq(WorkspaceFileEntity::getAgentId, agentId)
                        .isNull(WorkspaceFileEntity::getUserId)
                        .eq(WorkspaceFileEntity::getFilename, filename));
    }

    long size = content != null ? content.getBytes(StandardCharsets.UTF_8).length : 0;

    if (existing != null) {
        existing.setContent(content);
        existing.setFileSize(size);
        fileMapper.updateById(existing);
        eventPublisher.publishEvent(new WorkspaceFileChangedEvent(agentId, filename));
        return existing;
    } else {
        WorkspaceFileEntity entity = new WorkspaceFileEntity();
        entity.setAgentId(agentId);
        entity.setUserId(targetUserId);
        entity.setFilename(filename);
        entity.setContent(content);
        entity.setFileSize(size);
        entity.setEnabled(false);
        entity.setSortOrder(0);
        fileMapper.insert(entity);
        eventPublisher.publishEvent(new WorkspaceFileChangedEvent(agentId, filename));
        return entity;
    }
}

// 旧签名兼容
@Transactional
public WorkspaceFileEntity saveFile(Long agentId, String filename, String content) {
    return saveFile(agentId, filename, content, null);
}
```

#### 4.1.5 删除文件改造

```java
@Transactional
public void deleteFile(Long agentId, String filename, Long userId) {
    boolean isUserFile = !isGlobalFile(filename);
    Long targetUserId = (isUserFile && userId != null) ? userId : null;

    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId)
           .eq(WorkspaceFileEntity::getFilename, filename);
    if (targetUserId != null) {
        wrapper.eq(WorkspaceFileEntity::getUserId, targetUserId);
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }
    fileMapper.delete(wrapper);
    eventPublisher.publishEvent(new WorkspaceFileChangedEvent(agentId, filename));
}

// 旧签名兼容
@Transactional
public void deleteFile(Long agentId, String filename) {
    deleteFile(agentId, filename, null);
}
```

#### 4.1.6 系统提示构建改造

```java
/**
 * 将启用的工作区文件拼接为系统提示词（支持用户隔离）
 */
public String buildSystemPrompt(Long agentId, Long userId) {
    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId)
           .eq(WorkspaceFileEntity::getEnabled, true);
    if (userId != null) {
        wrapper.and(w -> w
                .isNull(WorkspaceFileEntity::getUserId)
                .or()
                .eq(WorkspaceFileEntity::getUserId, userId));
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }
    wrapper.orderByAsc(WorkspaceFileEntity::getSortOrder);

    List<WorkspaceFileEntity> enabledFiles = fileMapper.selectList(wrapper);

    if (enabledFiles.isEmpty()) {
        return null;
    }

    StringBuilder sb = new StringBuilder();
    for (WorkspaceFileEntity file : enabledFiles) {
        if (file.getContent() != null && !file.getContent().isBlank()) {
            if (!sb.isEmpty()) {
                sb.append("\n\n");
            }
            sb.append("--- ").append(file.getFilename()).append(" ---\n");
            sb.append(file.getContent().trim());
        }
    }
    return sb.isEmpty() ? null : sb.toString();
}

// 旧签名兼容
public String buildSystemPrompt(Long agentId) {
    return buildSystemPrompt(agentId, null);
}
```

#### 4.1.7 搜索方法改造

`searchSnippets` 方法也需要增加 `userId` 参数，在查询时过滤：

```java
public List<MemorySearchHit> searchSnippets(Long agentId, String query,
                                            Set<String> filenamePrefixes, int limit,
                                            Long userId) {
    // ... existing validation ...

    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId);
    
    // 增加用户隔离过滤
    if (userId != null) {
        wrapper.and(w -> w
                .isNull(WorkspaceFileEntity::getUserId)
                .or()
                .eq(WorkspaceFileEntity::getUserId, userId));
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }

    // ... rest of existing logic ...
}
```

#### 4.1.8 getPromptFiles 改造

```java
public List<String> getPromptFiles(Long agentId, Long userId) {
    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId)
           .eq(WorkspaceFileEntity::getEnabled, true);
    if (userId != null) {
        wrapper.and(w -> w
                .isNull(WorkspaceFileEntity::getUserId)
                .or()
                .eq(WorkspaceFileEntity::getUserId, userId));
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }
    wrapper.orderByAsc(WorkspaceFileEntity::getSortOrder);
    
    return fileMapper.selectList(wrapper)
            .stream()
            .map(WorkspaceFileEntity::getFilename)
            .distinct()
            .collect(Collectors.toList());
}

// 旧签名兼容
public List<String> getPromptFiles(Long agentId) {
    return getPromptFiles(agentId, null);
}
```

#### 4.1.9 setPromptFiles 改造

```java
@Transactional
public void setPromptFiles(Long agentId, List<String> filenames, Long userId) {
    // 只查询当前用户可见的文件（全局 + 用户私有）
    LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
    wrapper.eq(WorkspaceFileEntity::getAgentId, agentId);
    if (userId != null) {
        wrapper.and(w -> w
                .isNull(WorkspaceFileEntity::getUserId)
                .or()
                .eq(WorkspaceFileEntity::getUserId, userId));
    } else {
        wrapper.isNull(WorkspaceFileEntity::getUserId);
    }
    List<WorkspaceFileEntity> allFiles = fileMapper.selectList(wrapper);

    for (WorkspaceFileEntity file : allFiles) {
        int index = filenames.indexOf(file.getFilename());
        if (index >= 0) {
            file.setEnabled(true);
            file.setSortOrder(index);
        } else {
            file.setEnabled(false);
            file.setSortOrder(0);
        }
        fileMapper.updateById(file);
    }
}

// 旧签名兼容
@Transactional
public void setPromptFiles(Long agentId, List<String> filenames) {
    setPromptFiles(agentId, filenames, null);
}
```

## 5. Memory Provider 层改造

### 5.1 BuiltinMemoryProvider

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/provider/BuiltinMemoryProvider.java`

在 `systemPromptBlock` 中透传当前用户 ID：

```java
package vip.mate.memory.provider;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import vip.mate.agent.context.ChatOrigin;
import vip.mate.agent.context.ChatOriginHolder;
import vip.mate.memory.spi.MemoryProvider;
import vip.mate.workspace.document.WorkspaceFileService;

import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class BuiltinMemoryProvider implements MemoryProvider {

    private final WorkspaceFileService workspaceFileService;

    @Override
    public String id() {
        return "builtin";
    }

    @Override
    public int order() {
        return 0;
    }

    @Override
    public boolean isAvailable() {
        return true;
    }

    @Override
    public String systemPromptBlock(Long agentId) {
        try {
            Long userId = resolveCurrentUserId();
            String prompt = workspaceFileService.buildSystemPrompt(agentId, userId);
            return prompt != null ? prompt : "";
        } catch (Exception e) {
            log.warn("[BuiltinMemory] Failed to build system prompt for agent={}: {}",
                    agentId, e.getMessage());
            return "";
        }
    }

    @Override
    public String prefetch(Long agentId, String userQuery) {
        return "";
    }

    @Override
    public void syncTurn(Long agentId, String conversationId,
                         String userMessage, String assistantReply) {
    }

    @Override
    public List<Object> getToolBeans() {
        return List.of();
    }

    private Long resolveCurrentUserId() {
        ChatOrigin origin = ChatOriginHolder.get();
        if (origin == null || origin.requesterId() == null || origin.requesterId().isBlank()) {
            return null;
        }
        try {
            return Long.valueOf(origin.requesterId());
        } catch (NumberFormatException e) {
            return null;
        }
    }
}
```

### 5.2 StructuredMemoryProvider / StructuredMemoryService

**文件**：
- `mateclaw-server/src/main/java/vip/mate/memory/provider/StructuredMemoryProvider.java`
- `mateclaw-server/src/main/java/vip/mate/memory/service/StructuredMemoryService.java`

`StructuredMemoryService` 中的 `readFileSafe` 需要支持 userId：

```java
// StructuredMemoryService 中
private String readFileSafe(Long agentId, String filename, Long userId) {
    try {
        WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
        return file != null && file.getContent() != null ? file.getContent() : "";
    } catch (Exception e) {
        return "";
    }
}
```

> 注意：`StructuredMemoryProvider` 的接口签名 `systemPromptBlock(Long agentId)` 无法直接获取 userId。由于 `structured/*.md` 属于用户私有文件，需要通过 `ChatOriginHolder` 在 `StructuredMemoryProvider` 或 `StructuredMemoryService` 中解析当前用户。

**建议方案**：在 `StructuredMemoryService` 中统一从 `ChatOriginHolder` 获取 userId：

```java
@Service
@RequiredArgsConstructor
public class StructuredMemoryService {
    
    private final WorkspaceFileService workspaceFileService;
    private final ApplicationEventPublisher eventPublisher;


    private Long resolveUserId() {
        ChatOrigin origin = ChatOriginHolder.get();
        if (origin == null || origin.requesterId() == null) return null;
        try {
            return Long.valueOf(origin.requesterId());
        } catch (NumberFormatException e) {
            return null;
        }
    }

    private String readFileSafe(Long agentId, String filename) {
        Long userId = resolveUserId();
        try {
            WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
            return file != null && file.getContent() != null ? file.getContent() : "";
        } catch (Exception e) {
            return "";
        }
    }
}
```

### 5.3 MemoryRecallTracker

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/service/MemoryRecallTracker.java`

追踪召回时也需要按用户隔离：

```java
public void trackRecalls(Long agentId, String userQuery) {
    try {
        Long userId = resolveCurrentUserId();
        
        LambdaQueryWrapper<WorkspaceFileEntity> wrapper = new LambdaQueryWrapper<>();
        wrapper.eq(WorkspaceFileEntity::getAgentId, agentId)
               .eq(WorkspaceFileEntity::getEnabled, true)
               .isNotNull(WorkspaceFileEntity::getContent)
               .ne(WorkspaceFileEntity::getContent, "");
        
        if (userId != null) {
            wrapper.and(w -> w
                    .isNull(WorkspaceFileEntity::getUserId)
                    .or()
                    .eq(WorkspaceFileEntity::getUserId, userId));
        } else {
            wrapper.isNull(WorkspaceFileEntity::getUserId);
        }
        
        wrapper.orderByAsc(WorkspaceFileEntity::getSortOrder);
        List<WorkspaceFileEntity> injectedFiles = workspaceFileMapper.selectList(wrapper);
        
        // ... rest of existing logic ...
    } catch (Exception e) {
        log.warn("[MemoryRecall] Failed to track recalls for agent={}: {}", agentId, e.getMessage());
    }
}

private Long resolveCurrentUserId() {
    ChatOrigin origin = ChatOriginHolder.get();
    if (origin == null || origin.requesterId() == null) return null;
    try {
        return Long.valueOf(origin.requesterId());
    } catch (NumberFormatException e) {
        return null;
    }
}
```

## 6. Tool 层改造

### 6.1 WorkspaceMemoryTool

**文件**：`mateclaw-server/src/main/java/vip/mate/tool/builtin/WorkspaceMemoryTool.java`

所有工具方法需要从 `ChatOriginHolder` 获取当前用户 ID，并透传给 `WorkspaceFileService`：

```java
private Long resolveCurrentUserId() {
    ChatOrigin origin = ChatOriginHolder.get();
    if (origin == null || origin.requesterId() == null || origin.requesterId().isBlank()) {
        return null;
    }
    try {
        return Long.valueOf(origin.requesterId());
    } catch (NumberFormatException e) {
        return null;
    }
}
```

然后各工具方法内部：

```java
// read_workspace_memory_file
Long userId = resolveCurrentUserId();
WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);

// write_workspace_memory_file
Long userId = resolveCurrentUserId();
WorkspaceFileEntity before = workspaceFileService.getFile(agentId, filename, userId);
WorkspaceFileEntity saved = workspaceFileService.saveFile(agentId, filename, content, userId);

// edit_workspace_memory_file
Long userId = resolveCurrentUserId();
WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
// ... edit logic ...
workspaceFileService.saveFile(agentId, filename, updated, userId);

// list_workspace_memory_files
Long userId = resolveCurrentUserId();
List<WorkspaceFileEntity> files = workspaceFileService.listFiles(agentId, userId);

// search_workspace_memory
Long userId = resolveCurrentUserId();
List<MemorySearchHit> hits = workspaceFileService.searchSnippets(agentId, trimmed, scopes, limit, userId);
```

### 6.2 UniversalMemoryTool

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/tool/UniversalMemoryTool.java`

`remember` 方法追加到 `MEMORY.md`，属于全局文件，不需要 userId。但如果是追加到用户私有文件，则需要透传。

### 6.3 StructuredMemoryTool

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/tool/StructuredMemoryTool.java`

`remember_structured` / `recall_structured` / `forget_structured` 操作的是 `structured/*.md`（用户私有），需要确保 `StructuredMemoryService` 内部已支持 userId。

## 7. Controller 层改造

### 7.1 WorkspaceFileController

**文件**：`mateclaw-server/src/main/java/vip/mate/workspace/document/controller/WorkspaceFileController.java`

所有端点需要从 `Authentication` 解析 `userId` 并透传：

```java
@RestController
@RequestMapping("/api/v1/agents/{agentId}/workspace")
@RequiredArgsConstructor
public class WorkspaceFileController {

    private final WorkspaceFileService workspaceFileService;
    private final WorkspaceMemoryArchiveService memoryArchiveService;

    @GetMapping("/files")
    public R<List<WorkspaceFileEntity>> listFiles(@PathVariable Long agentId,
                                                   Authentication auth) {
        Long userId = resolveUserId(auth);
        return R.ok(workspaceFileService.listFiles(agentId, userId));
    }

    @GetMapping("/files/**")
    public R<WorkspaceFileEntity> getFile(@PathVariable Long agentId,
                                           HttpServletRequest request,
                                           Authentication auth) {
        String filename = extractFilename(request);
        Long userId = resolveUserId(auth);
        WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
        if (file == null) {
            return R.fail("文件不存在: " + filename);
        }
        return R.ok(file);
    }

    @PutMapping("/files/**")
    public R<WorkspaceFileEntity> saveFile(@PathVariable Long agentId,
                                            HttpServletRequest httpRequest,
                                            @RequestBody SaveFileRequest body,
                                            Authentication auth) {
        String filename = extractFilename(httpRequest);
        Long userId = resolveUserId(auth);
        return R.ok(workspaceFileService.saveFile(agentId, filename, body.getContent(), userId));
    }

    @DeleteMapping("/files/**")
    public R<Void> deleteFile(@PathVariable Long agentId,
                               HttpServletRequest request,
                               Authentication auth) {
        String filename = extractFilename(request);
        Long userId = resolveUserId(auth);
        workspaceFileService.deleteFile(agentId, filename, userId);
        return R.ok();
    }

    @GetMapping("/prompt-files")
    public R<List<String>> getPromptFiles(@PathVariable Long agentId,
                                           Authentication auth) {
        Long userId = resolveUserId(auth);
        return R.ok(workspaceFileService.getPromptFiles(agentId, userId));
    }

    @PutMapping("/prompt-files")
    public R<Void> setPromptFiles(@PathVariable Long agentId,
                                   @RequestBody PromptFilesRequest request,
                                   Authentication auth) {
        Long userId = resolveUserId(auth);
        workspaceFileService.setPromptFiles(agentId, request.getFiles(), userId);
        return R.ok();
    }

    private Long resolveUserId(Authentication auth) {
        if (auth == null || auth.getPrincipal() == null) {
            return null;
        }
        Object principal = auth.getPrincipal();
        if (principal instanceof vip.mate.core.security.UserDetailsImpl userDetails) {
            return userDetails.getId();
        }
        try {
            return Long.valueOf(auth.getName());
        } catch (NumberFormatException e) {
            return null;
        }
    }

    private String extractFilename(HttpServletRequest request) {
        String fullPath = (String) request.getAttribute(
                HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE);
        int filesIdx = fullPath.indexOf("/workspace/files/");
        return fullPath.substring(filesIdx + "/workspace/files/".length());
    }
}
```

### 7.2 MemoryController / DreamController

**文件**：
- `mateclaw-server/src/main/java/vip/mate/memory/controller/MemoryController.java`
- `mateclaw-server/src/main/java/vip/mate/memory/controller/DreamController.java`

涉及记忆摘要、HiL 编辑、Fact 操作的端点也需要透传 userId。特别是：

- `MemoryController.triggerSummarize` → `MemorySummarizationService.analyzeAndUpdateMemory` 需要知道当前用户
- `DreamController.editEntry` → `MemoryHilService.editMemoryEntry` 需要知道当前用户
- `FactController.forgetFact` 已有 auth 参数，需要确保 userId 透传到底层

## 8. AgentGraphBuilder Prompt 文案更新

**文件**：`mateclaw-server/src/main/java/vip/mate/agent/AgentGraphBuilder.java`

更新工具指导中的记忆文件说明，明确区分全局 vs 用户私有：

```java
String toolGuidance = """

        ## Runtime Context
        - Current Agent ID: %s

        ## Workspace Memory Guidelines
        Your durable memory is stored in database-backed workspace markdown files for this agent:

        Global shared files (same for all users):
        - `AGENTS.md`: agent behavior config, tool usage rules
        - `SOUL.md`: agent persona, tone, identity
        - `MEMORY.md`: distilled long-term memory, durable facts, lessons, recurring patterns

        User-private files (isolated per user):
        - `PROFILE.md`: stable user profile, preferences, collaboration style
        - `memory/YYYY-MM-DD.md`: daily notes, raw events, temporary observations, open loops
        - Other custom `.md` files

        Use workspace memory tools instead of local filesystem tools for those files:
        - `list_workspace_memory_files(agentId=..., filenamePrefix=...)`
        - `read_workspace_memory_file(agentId=..., filename=...)`
        - `write_workspace_memory_file(agentId=..., filename=..., content=...)`
        - `edit_workspace_memory_file(agentId=..., filename=..., oldText=..., newText=...)`

        Memory writing policy:
        - Stable user preference, identity, collaboration habit -> `PROFILE.md`
        - Stable project fact, workflow, tool setup, lesson learned, recurring decision -> `MEMORY.md`
        - One-off event, meeting note, temporary context, today's decision trace -> `memory/YYYY-MM-DD.md`
        - Read before write unless you are creating a brand new daily note
        - Do not store secrets or highly sensitive data unless the user explicitly asks
        - Updating workspace memory files is internal state maintenance for this agent and can be done proactively when useful
        """.formatted(entity.getId());
```

## 9. 其他需要改造的服务

### 9.1 MemorySummarizationService

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/service/MemorySummarizationService.java`

对话摘要后会更新 `PROFILE.md` / `MEMORY.md` / `memory/YYYY-MM-DD.md`。需要区分：
- `MEMORY.md` → 全局（无 userId）
- `PROFILE.md` → 用户私有（需要 userId）
- `memory/YYYY-MM-DD.md` → 用户私有（需要 userId）

在 `doAnalyzeAndUpdate` 中，读取现有记忆文件时需要传入当前用户 ID：

```java
private void doAnalyzeAndUpdate(Long agentId, String conversationId) {
    Long userId = resolveCurrentUserId();
    
    // 1. 加载对话消息
    List<MessageEntity> messages = conversationService.listMessages(conversationId);
    // ...

    // 2. 加载现有记忆文件内容（带用户隔离）
    String profileContent = readFileContentSafe(agentId, "PROFILE.md", userId);
    String memoryContent = readFileContentSafe(agentId, "MEMORY.md", null); // 全局
    String dailyFilename = "memory/" + LocalDate.now() + ".md";
    String dailyContent = readFileContentSafe(agentId, dailyFilename, userId);

    // ...
}

private String readFileContentSafe(Long agentId, String filename, Long userId) {
    try {
        WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
        return file != null && file.getContent() != null ? file.getContent() : "";
    } catch (Exception e) {
        return "";
    }
}
```

### 9.2 MemoryHilService

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/service/MemoryHilService.java`

HiL 编辑记忆条目时需要传入 userId：

```java
public void editMemoryEntry(Long agentId, String filename, String key, String newContent, Long userId) {
    WorkspaceFileEntity file = workspaceFileService.getFile(agentId, filename, userId);
    // ... edit logic ...
    workspaceFileService.saveFile(agentId, filename, updated, userId);
}
```

### 9.3 SoulSummarizerService

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/service/SoulSummarizerService.java`

SOUL.md 是全局文件，更新时不需要 userId。但读取 PROFILE.md / MEMORY.md 时需要：

```java
void updateSoul(Long agentId) {
    // SOUL.md 和 MEMORY.md 是全局的，PROFILE.md 是用户私有的
    // 由于 SOUL 更新是全局的，可能需要在无 userId 场景下运行，或明确指定 userId
    String memoryContent = readSafe(agentId, "MEMORY.md");
    String profileContent = readSafe(agentId, "PROFILE.md"); // 这里需要确定：SOUL 是否依赖用户 PROFILE？
    String currentSoul = readSafe(agentId, "SOUL.md");
    // ...
}
```

> 注：SOUL.md 作为全局人格文件，其更新逻辑可能需要特殊处理。如果 SOUL 需要融合所有用户的 PROFILE，则设计更复杂。建议初期保持 SOUL.md 仅基于全局 MEMORY.md 更新，不涉及用户私有 PROFILE。

### 9.4 MemoryNudgeService

**文件**：`mateclaw-server/src/main/java/vip/mate/memory/service/MemoryNudgeService.java`

`doNudge` 中加载 `structuredMemoryService.buildMemoryBlock(agentId)` 需要支持 userId。

### 9.5 WorkspaceMemoryArchiveService

**文件**：`mateclaw-server/src/main/java/vip/mate/workspace/document/WorkspaceMemoryArchiveService.java`

导出/导入记忆快照时，需要考虑：
- **导出**：导出当前用户的私有文件 + 全局文件？还是仅导出全局文件？
- **导入**：导入的文件按文件名自动分类归属（全局 vs 用户私有）

建议：导出时包含全局文件 + 当前用户私有文件；导入时按 `isGlobalFile()` 自动分配 `user_id`。

### 9.6 FactProjectionBuilder / FactController

**文件**：
- `mateclaw-server/src/main/java/vip/mate/memory/fact/projection/FactProjectionBuilder.java`
- `mateclaw-server/src/main/java/vip/mate/memory/fact/controller/FactController.java`

Fact 投影基于 `structured/*.md` 构建，属于用户私有文件。`FactProjectionBuilder.rebuildOne` 需要传入 userId。

## 10. 前端改造

### 10.1 API 调用层

前端 `agentContextApi` 不需要大改，因为 userId 是从后端 `Authentication` 自动解析的，不需要前端显式传递。

### 10.2 UI 展示

在 Agent 上下文页面（`AgentContext.vue`），文件列表需要展示文件归属标识：
- 全局文件：显示 "全局共享" 标签
- 用户私有文件：显示 "个人私有" 标签

### 10.3 新增/编辑文件时的归属提示

当用户创建新文件时