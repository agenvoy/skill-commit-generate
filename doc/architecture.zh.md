# commit-generate - 架構

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    User[使用者呼叫 /commit-generate] --> Skill[SKILL.md 規則]
    Skill --> Input[輸入層<br/>並行讀取四份資料]
    Input --> Diff[git diff --cached<br/>內容來源]
    Input --> Status[git status --short<br/>漏 stage 提醒]
    Input --> Context[git log / branch<br/>用詞參考]
    Diff --> Detect[跨主題偵測]
    Detect --> Upgrade[Tag 升級訊號]
    Upgrade --> Tags[Tag 優先序]
    Tags --> Format[輸出格式規則]
    Status --> Output[雙語 commit message]
    Context --> Format
    Format --> Output
```

## Module: 輸入層

同一輪並行讀取四份資料；diff 為空時直接報錯停止，不回退到工作區。

```mermaid
graph TB
    subgraph Input[輸入層]
        A[git diff --cached] --> B{輸出為空?}
        S[git status --short] --> R[比對同模組未 stage 檔案]
        L[git log --oneline -10] --> W[用詞參考]
        Br[git branch --show-current] --> W
        B -->|是| C[輸出錯誤訊息]
        B -->|否| D[交給跨主題偵測]
    end
    C --> Stop[停止]
    R --> Reminder[message 前的提醒行]
    W --> Wording[模組稱呼與措辭]
```

## Module: 跨主題偵測

判斷單次 diff 是否混雜無關意圖，命中則輸出拆分建議。

```mermaid
graph TB
    subgraph MultiTopic[跨主題偵測]
        A[接收 diff] --> B{觸及 2 個以上主要 Tag?}
        A --> C{橫跨 2 個以上無關模組?}
        A --> D{包含 3 個以上無關主題?}
        B -->|任一為是| E[標記為跨主題]
        C -->|任一為是| E
        D -->|任一為是| E
        B -->|皆為否| F[標記為單一主題]
        C -->|皆為否| F
        D -->|皆為否| F
    end
    E --> SplitWarn[輸出拆分建議]
    F --> Upgrade[Tag 升級掃描]
    SplitWarn --> Upgrade
```

## Module: Tag 升級訊號

由上而下掃描訊號，命中即強制升級並禁止降級為 `feat` 或 `update`。

```mermaid
graph TB
    subgraph Upgrade[Tag 升級訊號]
        A[接收 diff] --> B{命中 Breaking 訊號?}
        B -->|是| C[Tag = breaking]
        B -->|否| D{命中 Security 訊號?}
        D -->|是| E[Tag = security]
        D -->|否| F[依 Tag 優先序匹配意圖]
    end
    C --> Out[輸出 Tag]
    E --> Out
    F --> Out
```

## Module: 輸出格式

以單一 Tag 組出英文 subject 與繁體中文 body。

```mermaid
graph LR
    subgraph Format[輸出格式]
        T[Tag] --> EN[英文 subject<br/>imperative、≤ 72 字元]
        T --> ZH[繁體中文 body<br/>動詞開頭、≤ 50 字]
    end
    EN --> Msg[tag: English description<br/>tag: 中文描述]
    ZH --> Msg
```

## 資料流

```mermaid
sequenceDiagram
    participant User as 使用者
    participant Agent as Agent Harness
    participant Skill as SKILL.md
    participant Git

    User->>Git: git add <files>
    User->>Agent: /commit-generate
    Agent->>Skill: 載入 skill 定義
    par 並行讀取
        Skill->>Git: git diff --cached
        Skill->>Git: git status --short
        Skill->>Git: git log --oneline -10
        Skill->>Git: git branch --show-current
    end
    Git-->>Skill: 四份結果
    alt diff 為空
        Skill-->>User: 目前沒有 staged 變更
    else diff 非空
        Skill->>Skill: 跨主題偵測
        Skill->>Skill: Tag 升級掃描
        Skill->>Skill: 套用輸出格式
        opt 同模組有未 stage 檔案
            Skill-->>User: 提醒行
        end
        opt 跨主題
            Skill-->>User: 拆分建議
        end
        Skill-->>User: 雙語 commit message
    end
```

## Tag 決策狀態機

```mermaid
stateDiagram-v2
    [*] --> ScanBreaking
    ScanBreaking --> Breaking: 命中 Breaking 訊號
    ScanBreaking --> ScanSecurity: 未命中
    ScanSecurity --> Security: 命中 Security 訊號
    ScanSecurity --> MatchIntent: 未命中
    MatchIntent --> Resolved: 依優先序選出 Tag
    Breaking --> Resolved
    Security --> Resolved
    Resolved --> [*]
```
