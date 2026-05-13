<%*
const sourceFile = tp.config.target_file;
let originalContent = await app.vault.read(sourceFile);

const fs = window.require("fs");
const path = window.require("path");

function escapeYaml(value) {
  return String(value)
    .replace(/\\/g, "\\\\")
    .replace(/"/g, '\\"');
}

function sanitizeFileName(value) {
  return String(value)
    .trim()
    .replace(/[\\/:*?"<>|]/g, "")
    .replace(/\s+/g, "_")
    .replace(/_+/g, "_")
    .replace(/^_|_$/g, "");
}

// 기존 YAML Front Matter 제거
originalContent = originalContent.replace(/^---\s*\n[\s\S]*?\n---\s*\n*/, "").trim();

// 문서 제목 추출: 첫 번째 H1 우선, 없으면 원본 파일명 사용
let title = sourceFile.basename;
const headingMatch = originalContent.match(/^#\s+(.+)\s*$/m);

if (headingMatch) {
  title = headingMatch[1].trim();

  // 본문에서 첫 번째 H1 제거
  originalContent = originalContent.replace(/^#\s+(.+)\s*\n*/, "").trim();
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
// 공백은 '_'로 치환
const baseFileName = sanitizeFileName(sourceFile.basename);

const date = tp.date.now("YYYY-MM-DD");
const lastModifiedAt = tp.date.now("YYYY-MM-DD HH:mm:ss");

const outputFileName = `${date}-note_${baseFileName}.md`;
const outputPath = path.join(outputFolderPath, outputFileName);

const newContent = `---
layout: single
title: "${escapeYaml(title)}"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: ${date}
last_modified_at: ${lastModifiedAt}
---

# ${title}

${originalContent}

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