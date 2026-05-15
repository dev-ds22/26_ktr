---
layout: single
title: "Jekyll-template Sample"
excerpt: "Jekyll-template Sample"

categories:
  - tech
tags:
  - [tech, memo]

toc: false
toc_sticky: true

date: "2026-05-14"
last_modified_at: "2026-05-14 14:21:59"
---

## jekyll 용 post 생성 Script
```javascript
<%*
const sourceFile = tp.config.target_file;
let originalContent = await app.vault.read(sourceFile);

const fs = window.require("fs");
const path = window.require("path");

const DEFAULT_CATEGORIES = ["tech"];
const DEFAULT_TAGS = ["tech", "memo"];

function escapeYaml(value) {
  return String(value || "")
    .replace(/\\/g, "\\\\")
    .replace(/"/g, '\\"');
}

function normalizeText(value) {
  return String(value || "").trim();
}

function sanitizeFileName(value) {
  let name = String(value || "untitled")
    .trim()
    .normalize("NFC")
    .replace(/\./g, "_")
    .replace(/[\\/:*?"<>|\x00-\x1F]/g, "_")
    .replace(/\s+/g, "_")
    .replace(/_+/g, "_")
    .replace(/^_+|_+$/g, "");

  if (!name) {
    name = "untitled";
  }

  if (/^(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])$/i.test(name)) {
    name = `_${name}`;
  }

  return name;
}

function sanitizeTitleFromFileName(value) {
  let title = String(value || "")
    .trim()
    .normalize("NFC")
    .replace(/\./g, "_")
    .replace(/[\\/:*?"<>|\x00-\x1F]/g, "_")
    .replace(/\s+/g, "_")
    .replace(/_+/g, "_")
    .replace(/^_+|_+$/g, "");

  return title;
}

function removeFencedCodeBlocks(content) {
  return String(content || "").replace(/```[\s\S]*?```/g, "");
}

function findFirstH1OutsideCodeBlock(content) {
  const contentWithoutCodeBlock = removeFencedCodeBlocks(content);
  const headingMatch = contentWithoutCodeBlock.match(/^#\s+(.+)\s*$/m);

  if (!headingMatch) {
    return "";
  }

  return headingMatch[1].trim();
}

function removeFirstH1OutsideCodeBlock(content, headingText) {
  const lines = String(content || "").split(/\r?\n/);
  let inCodeBlock = false;
  let removed = false;
  const result = [];

  for (const line of lines) {
    if (/^\s*```/.test(line)) {
      inCodeBlock = !inCodeBlock;
      result.push(line);
      continue;
    }

    if (!removed && !inCodeBlock) {
      const match = line.match(/^#\s+(.+)\s*$/);
      if (match && normalizeText(match[1]) === normalizeText(headingText)) {
        removed = true;
        continue;
      }
    }

    result.push(line);
  }

  return result.join("\n").trim();
}

function extractFrontMatter(content) {
  const match = content.match(/^---\s*\r?\n([\s\S]*?)\r?\n---\s*\r?\n*/);

  if (!match) {
    return {
      frontMatterText: "",
      body: content
    };
  }

  return {
    frontMatterText: match[1],
    body: content.slice(match[0].length)
  };
}

function extractYamlValueBlock(frontMatterText, key) {
  const lines = String(frontMatterText || "").split(/\r?\n/);
  const result = [];
  let found = false;
  let inlineValue = "";

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    const keyMatch = line.match(new RegExp(`^${key}\\s*:\\s*(.*)$`));

    if (keyMatch) {
      found = true;
      inlineValue = keyMatch[1].trim();

      if (inlineValue) {
        return [inlineValue];
      }

      continue;
    }

    if (found) {
      if (/^\S/.test(line) && !line.trim().startsWith("-")) {
        break;
      }

      if (/^\s*-\s+/.test(line)) {
        result.push(line.replace(/^\s*-\s+/, "").trim());
      } else if (line.trim()) {
        result.push(line.trim());
      }
    }
  }

  return result;
}

function parseYamlList(frontMatterText, key, defaultValues) {
  const rawValues = extractYamlValueBlock(frontMatterText, key);

  if (!rawValues || rawValues.length === 0) {
    return defaultValues;
  }

  let values = [];

  rawValues.forEach((raw) => {
    let text = String(raw || "").trim();

    if (!text) return;

    text = text.replace(/^['"]|['"]$/g, "");

    if (text.startsWith("[") && text.endsWith("]")) {
      text = text.slice(1, -1);
      values.push(
        ...text
          .split(",")
          .map((v) => v.trim().replace(/^['"]|['"]$/g, ""))
          .filter(Boolean)
      );
      return;
    }

    if (text.includes(",")) {
      values.push(
        ...text
          .split(",")
          .map((v) => v.trim().replace(/^['"]|['"]$/g, ""))
          .filter(Boolean)
      );
      return;
    }

    values.push(text);
  });

  values = values
    .map((v) => String(v || "").trim())
    .filter(Boolean);

  return values.length > 0 ? values : defaultValues;
}

function toYamlList(key, values) {
  return `${key}:\n${values.map((v) => `  - "${escapeYaml(v)}"`).join("\n")}`;
}

// 원본 YAML Front Matter 추출
const frontMatterInfo = extractFrontMatter(originalContent);
const sourceFrontMatterText = frontMatterInfo.frontMatterText;

// 기존 YAML Front Matter 제거 후 본문만 사용
originalContent = frontMatterInfo.body.trim();

// 원본 MD에서 categories, tags 동적 추출
const categories = parseYamlList(sourceFrontMatterText, "categories", DEFAULT_CATEGORIES);
const tags = parseYamlList(sourceFrontMatterText, "tags", DEFAULT_TAGS);

// 파일명은 원본 MD 파일명을 기준으로 생성
// '.', 파일명 부적합 문자, 공백은 '_'로 치환
const baseFileName = sanitizeFileName(sourceFile.basename);

// title은 원본 파일명의 변환값을 우선 사용
// 원본 파일명으로 title을 만들 수 없을 경우에만 본문 H1을 사용
let title = sanitizeTitleFromFileName(sourceFile.basename);
let firstHeading = "";
let hideFirstHeading = false;

if (!title) {
  firstHeading = findFirstH1OutsideCodeBlock(originalContent);

  if (firstHeading) {
    title = firstHeading;
  } else {
    title = "untitled";
  }
} else {
  firstHeading = findFirstH1OutsideCodeBlock(originalContent);
}

// title과 원문 첫 번째 H1 제목이 같으면 본문에서 H1 제거
// 단, ``` 코드블록 안의 # 문구는 대상에서 제외
if (firstHeading && normalizeText(title) === normalizeText(firstHeading)) {
  originalContent = removeFirstH1OutsideCodeBlock(originalContent, firstHeading);
  hideFirstHeading = true;
}

// 현재 Vault 실제 OS 경로 취득
const vaultBasePath = app.vault.adapter.getBasePath();

// Vault 한 단계 상위 폴더
const vaultParentPath = path.dirname(vaultBasePath);

// Vault 한 단계 상위의 _posts 폴더
const outputFolderPath = path.join(vaultParentPath, "_posts");

// _posts 폴더 없으면 생성
if (!fs.existsSync(outputFolderPath)) {
  fs.mkdirSync(outputFolderPath, { recursive: true });
}

const dateOnly = tp.date.now("YYYY-MM-DD");
const dateTime = tp.date.now("YYYY-MM-DD HH:mm:ss") + " +0900";

const outputFileName = `${dateOnly}-note_${baseFileName}.md`;
const outputPath = path.join(outputFolderPath, outputFileName);

// const headingBlock = hideFirstHeading ? "" : `# ${title}\n\n`;
const headingBlock = "";

const newContent = `---
layout: single
title: "${escapeYaml(title)}"
excerpt: "${escapeYaml(title)}"
${toYamlList("categories", categories)}
${toYamlList("tags", tags)}
toc: false
toc_sticky: true
date: "${dateOnly}"
last_modified_at: "${dateTime}"
---
${headingBlock}${originalContent}
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
`;

// 동일 파일명이 있으면 덮어쓰기
fs.writeFileSync(outputPath, newContent, "utf8");

new Notice(`Jekyll 변환 파일 생성/덮어쓰기 완료: ${outputPath}`);
-%>
```


정확도: 96%

## 변경 반영 완료 스크립트

아래 스크립트는 요청하신 2가지를 반영한 버전입니다.

```text
1. output 파일명 생성 시 '.', 파일명 부적합 문자 → '_' 변환
2. 원본 MD Front Matter의 categories, tags 값을 읽어서 Jekyll output에 동적 반영
```

```javascript id="jekyll-templater-script"
<%*
const sourceFile = tp.config.target_file;
let originalContent = await app.vault.read(sourceFile);
const fs = window.require("fs");
const path = window.require("path");
const DEFAULT_CATEGORIES = ["tech"];
const DEFAULT_TAGS = ["tech", "memo"];
function escapeYaml(value) {
  return String(value || "")
    .replace(/\\/g, "\\\\")
    .replace(/"/g, '\\"');
}
function normalizeText(value) {
  return String(value || "").trim();
}
function sanitizeFileName(value) {
  let name = String(value || "untitled")
    .trim()
    .normalize("NFC")
    .replace(/\./g, "_")
    .replace(/[\\/:*?"<>|\x00-\x1F]/g, "_")
    .replace(/\s+/g, "_")
    .replace(/_+/g, "_")
    .replace(/^_+|_+$/g, "");
  if (!name) {
    name = "untitled";
  }
  if (/^(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])$/i.test(name)) {
    name = `_${name}`;
  }
  return name;
}
function extractFrontMatter(content) {
  const match = content.match(/^---\s*\r?\n([\s\S]*?)\r?\n---\s*\r?\n*/);
  if (!match) {
    return {
      frontMatterText: "",
      body: content
    };
  }
  return {
    frontMatterText: match[1],
    body: content.slice(match[0].length)
  };
}
function extractYamlValueBlock(frontMatterText, key) {
  const lines = String(frontMatterText || "").split(/\r?\n/);
  const result = [];
  let found = false;
  let inlineValue = "";
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    const keyMatch = line.match(new RegExp(`^${key}\\s*:\\s*(.*)$`));
    if (keyMatch) {
      found = true;
      inlineValue = keyMatch[1].trim();
      if (inlineValue) {
        return [inlineValue];
      }
      continue;
    }
    if (found) {
      if (/^\S/.test(line) && !line.trim().startsWith("-")) {
        break;
      }
      if (/^\s*-\s+/.test(line)) {
        result.push(line.replace(/^\s*-\s+/, "").trim());
      } else if (line.trim()) {
        result.push(line.trim());
      }
    }
  }
  return result;
}
function parseYamlList(frontMatterText, key, defaultValues) {
  const rawValues = extractYamlValueBlock(frontMatterText, key);
  if (!rawValues || rawValues.length === 0) {
    return defaultValues;
  }
  let values = [];
  rawValues.forEach((raw) => {
    let text = String(raw || "").trim();
    if (!text) return;
    text = text.replace(/^['"]|['"]$/g, "");
    if (text.startsWith("[") && text.endsWith("]")) {
      text = text.slice(1, -1);
      values.push(
        ...text
          .split(",")
          .map((v) => v.trim().replace(/^['"]|['"]$/g, ""))
          .filter(Boolean)
      );
      return;
    }
    if (text.includes(",")) {
      values.push(
        ...text
          .split(",")
          .map((v) => v.trim().replace(/^['"]|['"]$/g, ""))
          .filter(Boolean)
      );
      return;
    }
    values.push(text);
  });
  values = values
    .map((v) => String(v || "").trim())
    .filter(Boolean);
  return values.length > 0 ? values : defaultValues;
}
function toYamlList(key, values) {
  return `${key}:\n${values.map((v) => `  - "${escapeYaml(v)}"`).join("\n")}`;
}
// 원본 YAML Front Matter 추출
const frontMatterInfo = extractFrontMatter(originalContent);
const sourceFrontMatterText = frontMatterInfo.frontMatterText;
// 기존 YAML Front Matter 제거 후 본문만 사용
originalContent = frontMatterInfo.body.trim();
// 원본 MD에서 categories, tags 동적 추출
const categories = parseYamlList(sourceFrontMatterText, "categories", DEFAULT_CATEGORIES);
const tags = parseYamlList(sourceFrontMatterText, "tags", DEFAULT_TAGS);
// 원문 첫 번째 H1 제목 추출
let title = sourceFile.basename;
let firstHeading = "";
let hideFirstHeading = false;
const headingMatch = originalContent.match(/^#\s+(.+)\s*$/m);
if (headingMatch) {
  firstHeading = headingMatch[1].trim();
  title = firstHeading;
  // title과 원문 첫 번째 # 제목이 같으면 본문에서 H1 제거
  if (normalizeText(title) === normalizeText(firstHeading)) {
    originalContent = originalContent.replace(/^#\s+(.+)\s*\n*/, "").trim();
    hideFirstHeading = true;
  }
}
// 현재 Vault 실제 OS 경로 취득
const vaultBasePath = app.vault.adapter.getBasePath();
// Vault 한 단계 상위 폴더
const vaultParentPath = path.dirname(vaultBasePath);
// Vault 한 단계 상위의 _posts 폴더
const outputFolderPath = path.join(vaultParentPath, "_posts");
// _posts 폴더 없으면 생성
if (!fs.existsSync(outputFolderPath)) {
  fs.mkdirSync(outputFolderPath, { recursive: true });
}
// 파일명은 원본 MD 파일명을 기준으로 생성
// '.', 파일명 부적합 문자, 공백은 '_'로 치환
const baseFileName = sanitizeFileName(sourceFile.basename);
const dateOnly = tp.date.now("YYYY-MM-DD");
const dateTime = tp.date.now("YYYY-MM-DD HH:mm:ss") + " +0900";
const outputFileName = `${dateOnly}-note_${baseFileName}.md`;
const outputPath = path.join(outputFolderPath, outputFileName);
// const headingBlock = hideFirstHeading ? "" : `# ${title}\n\n`;
const headingBlock = "";
const newContent = `---
layout: single
title: "${escapeYaml(title)}"
excerpt: "${escapeYaml(title)}"
${toYamlList("categories", categories)}
${toYamlList("tags", tags)}
toc: false
toc_sticky: true
date: "${dateOnly}"
last_modified_at: "${dateTime}"
---
${headingBlock}${originalContent}
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
`;
// 동일 파일명이 있으면 덮어쓰기
fs.writeFileSync(outputPath, newContent, "utf8");
new Notice(`Jekyll 변환 파일 생성/덮어쓰기 완료: ${outputPath}`);
-%>
```

## 원본 MD 작성 예시

### 배열 방식

```markdown id="source-md-array"
---
categories: [spring, architecture]
tags: [spring, cache, performance]
---
# Spring Cache 정리
본문 내용
```

### YAML 리스트 방식

```markdown id="source-md-list"
---
categories:
  - spring
  - architecture
tags:
  - spring
  - cache
  - performance
---
# Spring Cache 정리
본문 내용
```

### 콤마 방식

```markdown id="source-md-comma"
---
categories: spring, architecture
tags: spring, cache, performance
---
# Spring Cache 정리
본문 내용
```

## 변경 후 output 예시

원본 파일명이 아래라면:

```text
Spring 5.3 / Cache.test?.md
```

생성 파일명은 아래처럼 됩니다.

```text
2026-05-15-note_Spring_5_3_Cache_test.md
```

생성되는 Jekyll Front Matter는 아래처럼 됩니다.

```yaml id="output-frontmatter"
---
layout: single
title: "Spring Cache 정리"
excerpt: "Spring Cache 정리"
categories:
  - "spring"
  - "architecture"
tags:
  - "spring"
  - "cache"
  - "performance"
toc: false
toc_sticky: true
date: "2026-05-15"
last_modified_at: "2026-05-15 11:00:00 +0900"
---
```

## 핵심 수정 포인트

```javascript id="key-change"
.replace(/\./g, "_")
.replace(/[\\/:*?"<>|\x00-\x1F]/g, "_")
```

기존에는 파일명 부적합 문자를 제거했지만, 변경 후에는 `_`로 변환합니다.
또한 기존:

```yaml id="old"
categories:
  - tech
tags:
  - [tech, memo]
```

를 고정 출력하지 않고, 원본 MD의 `categories`, `tags` 값을 읽어 아래처럼 정상 리스트로 출력합니다.

```yaml id="new"
categories:
  - "spring"
tags:
  - "spring"
  - "cache"
```

원본 MD에 `categories`, `tags`가 없으면 기존 기본값인 `tech`, `tech/memo`가 적용됩니다.



<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
