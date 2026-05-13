<%*
const sourceFile = tp.config.target_file;
let originalContent = await app.vault.read(sourceFile);

// output 폴더명
const outputFolder = "output";

// YAML 문자열 이스케이프
function escapeYaml(value) {
  return String(value)
    .replace(/\\/g, "\\\\")
    .replace(/"/g, '\\"');
}

// Windows/파일시스템에서 문제 되는 문자 제거
function sanitizeFileName(value) {
  return String(value)
    .trim()
    .replace(/[\\/:*?"<>|]/g, "")
    .replace(/\s+/g, "-");
}

// 기존 YAML Front Matter 제거
originalContent = originalContent.replace(/^---\s*\n[\s\S]*?\n---\s*\n*/, "").trim();

// 문서 첫 번째 H1 제목 추출, 없으면 파일명 사용
let title = tp.file.title;
const headingMatch = originalContent.match(/^#\s+(.+)\s*$/m);

if (headingMatch) {
  title = headingMatch[1].trim();

  // 본문에서 첫 번째 H1 제거
  originalContent = originalContent.replace(/^#\s+(.+)\s*\n*/, "").trim();
}

const date = tp.date.now("YYYY-MM-DD");
const lastModifiedAt = tp.date.now("YYYY-MM-DD HH:mm:ss");
const safeTitle = sanitizeFileName(title);

// output 폴더 생성
if (!app.vault.getAbstractFileByPath(outputFolder)) {
  await app.vault.createFolder(outputFolder);
}

// 기본 output 파일 경로
let outputPath = `${outputFolder}/${date}-${safeTitle}.md`;

// 동일 파일명 존재 시 중복 방지
let index = 1;
while (app.vault.getAbstractFileByPath(outputPath)) {
  outputPath = `${outputFolder}/${date}-${safeTitle}-${index}.md`;
  index++;
}

// 변환 파일 내용
const newContent = `---
layout: single
title: "${escapeYaml(title)}"
excerpt: "${escapeYaml(title)}"

categories:
  - tech
tags:
  - [tech]

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

// 별도 파일 생성
const createdFile = await app.vault.create(outputPath, newContent);

// 생성된 파일 열기
await app.workspace.getLeaf(true).openFile(createdFile);

new Notice(`Jekyll 변환 파일 생성 완료: ${outputPath}`);
-%>