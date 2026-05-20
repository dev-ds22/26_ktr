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
function sanitizeDirectoryName(value, maxLength) {
  let name = sanitizeFileName(value);
  if (maxLength && name.length > maxLength) {
    name = name.substring(0, maxLength).replace(/^_+|_+$/g, "");
  }
  if (!name) {
    name = "untitled";
  }
  return name;
}
function sanitizeAssetFileName(fileName) {
  const ext = path.extname(fileName);
  const base = path.basename(fileName, ext);
  const safeBase = sanitizeFileName(base);
  const safeExt = String(ext || "")
    .toLowerCase()
    .replace(/[\\/:*?"<>|\x00-\x1F]/g, "_");
  return `${safeBase}${safeExt}`;
}
function sanitizeTitleFromFileName(value) {
  return sanitizeFileName(value);
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
function getSourceFileDirectoryPath() {
  const sourceRelativePath = sourceFile.path;
  const sourceRelativeDir = path.dirname(sourceRelativePath);
  const vaultBasePath = app.vault.adapter.getBasePath();
  if (!sourceRelativeDir || sourceRelativeDir === ".") {
    return vaultBasePath;
  }
  return path.join(vaultBasePath, sourceRelativeDir);
}
function findObsidianImageEmbeds(content) {
  const embeds = [];
  const regex = /!\[\[([^\]]+\.(?:png|jpg|jpeg|gif|webp|svg))(?:\|[^\]]*)?\]\]/gi;
  let match;
  while ((match = regex.exec(content)) !== null) {
    const fullMatch = match[0];
    const rawTarget = match[1].trim();
    const imageFileName = path.basename(rawTarget);
    embeds.push({
      fullMatch,
      rawTarget,
      imageFileName
    });
  }
  return embeds;
}
function copyImageToTargetFolder(sourceImagePath, targetImageFolderPath, targetFileName) {
  if (!fs.existsSync(sourceImagePath)) {
    return false;
  }
  if (!fs.existsSync(targetImageFolderPath)) {
    fs.mkdirSync(targetImageFolderPath, { recursive: true });
  }
  const targetImagePath = path.join(targetImageFolderPath, targetFileName);
  fs.copyFileSync(sourceImagePath, targetImagePath);
  return true;
}
function convertObsidianImagesToJekyllMarkdown(content, sourceImageDirPath, targetImageFolderPath, imageUrlDirName) {
  const embeds = findObsidianImageEmbeds(content);
  const copied = [];
  const missing = [];
  let convertedContent = content;
  embeds.forEach((embed) => {
    const originalImageFileName = embed.imageFileName;
    const safeImageFileName = sanitizeAssetFileName(originalImageFileName);
    const sourceImagePath = path.join(sourceImageDirPath, originalImageFileName);
    const copiedOk = copyImageToTargetFolder(
      sourceImagePath,
      targetImageFolderPath,
      safeImageFileName
    );
    if (copiedOk) {
      copied.push({
        from: sourceImagePath,
        to: path.join(targetImageFolderPath, safeImageFileName)
      });
      const altText = path.basename(safeImageFileName, path.extname(safeImageFileName));
      const markdownImage = `![${altText}](./../../images/${imageUrlDirName}/${safeImageFileName})`;
      convertedContent = convertedContent.split(embed.fullMatch).join(markdownImage);
    } else {
      missing.push(sourceImagePath);
    }
  });
  return {
    convertedContent,
    copied,
    missing
  };
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
// images 폴더는 _posts와 같은 레벨에 생성
const imagesRootFolderPath = path.join(vaultParentPath, "images");
// 생성 MD 파일명 기준 이미지 디렉토리명 생성
// 확장자 .md는 제외하고, 최대 40자까지만 사용
const outputFileNameWithoutExt = path.basename(outputFileName, ".md");
const imageDirName = sanitizeDirectoryName(outputFileNameWithoutExt, 40);
const outputImageFolderPath = path.join(imagesRootFolderPath, imageDirName);
if (!fs.existsSync(outputImageFolderPath)) {
  fs.mkdirSync(outputImageFolderPath, { recursive: true });
}
// Obsidian 이미지 파일은 원본 MD 파일과 같은 디렉토리에서 찾음
const sourceImageDirPath = getSourceFileDirectoryPath();
// Obsidian 이미지 문법 변환 + 이미지 파일 복사
const imageConvertResult = convertObsidianImagesToJekyllMarkdown(
  originalContent,
  sourceImageDirPath,
  outputImageFolderPath,
  imageDirName
);
originalContent = imageConvertResult.convertedContent;
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
let noticeMessage = `Jekyll 변환 파일 생성/덮어쓰기 완료: ${outputPath}`;
if (imageConvertResult.copied.length > 0) {
  noticeMessage += ` / 이미지 ${imageConvertResult.copied.length}개 복사: ${outputImageFolderPath}`;
}
if (imageConvertResult.missing.length > 0) {
  noticeMessage += ` / 이미지 ${imageConvertResult.missing.length}개 누락`;
  console.warn("[Jekyll 변환] 복사하지 못한 이미지:", imageConvertResult.missing);
}
new Notice(noticeMessage);
-%>