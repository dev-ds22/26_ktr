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