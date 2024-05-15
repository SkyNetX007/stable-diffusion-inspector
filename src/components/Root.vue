<template>
  <div class="text-center p-6 max-w-1024px mx-auto">
    <h1 class="font-bold text-3xl">NAI隐匿水印修改器</h1>
    <p class="text-gray-500 my-2 text-sm">
      修改NAI生成图像中的隐匿水印内容
    </p>
    <div v-if="imgFileRef" class="my-6">
      <div class="bg-white max-w-720px mx-auto border border-gray-300 p-2" v-if="imageRef">
        <img v-if="imageRef" v-bind="imageRef" alt="" style="display: block; width: auto; height: 30vh; margin:auto" />
      </div>
    </div>

    <div class="max-w-740px" style="margin: 0 auto">
      <el-upload class="upload-demo" drag multiple :before-upload="handleUpload">
        <el-icon class="el-icon--upload"><upload-filled /></el-icon>
        <div class="el-upload__text">拖动文件到这里或者点击选择文件</div>
      </el-upload>
    </div>

    <div v-if="imgFileRef" class="my-6">
      <div v-if="imgfileInfoRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">图片信息</h1>
        <div :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in imgfileInfoRef" :key="item.key">
          <h1 class="font-semibold text-sm text-gray-800">
            {{ item.key }}
            <el-popover placement="top-start" trigger="hover" content="点击复制" style="min-width: 10px"
              v-if="showCopyBtn(item.key)">
              <template #reference>
                <el-button style="margin-left: 6px" :icon="CopyDocument" :link="true"
                  @click="item.key == 'Comment' ? copy(jsonData.uc) : copy(item.value)" />
              </template>
            </el-popover>
          </h1>
          <el-input v-model="item.value" type="textarea" class="text-wrap break-all text-sm mt-1 text-gray-600"
            style="white-space: pre-wrap" />
        </div>
      </div>

      <div class="mt-4 text-left max-w-740px mx-auto">
        <el-button type="primary" @click="saveMetadata">保存元信息到隐匿水印</el-button>
      </div>

      <div v-if="exifRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">EXIF</h1>
        <div :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in exifRef" :key="item.key">
          <h1 class="font-semibold text-sm text-gray-800">{{ item.key }}</h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600" style="white-space: pre-wrap">
            {{ item.value.description }}
          </p>
        </div>
      </div>
    </div>

    <p class="text-gray-500 my-2 text-sm">
      *运算完全在你的设备上运行不会上传到云端
    </p>
    <div class="my-4 pt-4">
      <a class="inline-block text-sm text-gray-500"
        href="https://github.com/SkyNetX007/stable-diffusion-inspector">GitHub</a>
      <br />
    </div>
  </div>
</template>

<style>
.jv-container {
  line-height: 1.2;
}

.jv-code {
  padding: 10px 20px !important;
}
</style>

<script setup lang="ts">
import { ElMessage } from "element-plus";
import { ref } from "vue";
import useClipboard from "vue-clipboard3";
import { UploadFilled, CopyDocument } from "@element-plus/icons-vue";

import { asyncFileReaderAsDataURL, getStealthExif, embedStealthExif } from "../utils";

const imgFileRef = ref(null);
const imageRef = ref(null);
const exifRef = ref(null);
const imgfileInfoRef = ref(null);

const modelFileRef = ref(null);
const modelFileInfoRef = ref(null);

const jsonData = ref(null);
const imageMaxSizeRef = ref(0);
const { toClipboard } = useClipboard();

const availableImgExt = ["png", "jpeg", "jpg", "webp", "bmp"]

const templateMetadata = { Description: "", Software: "NovelAI", Source: "Stable Diffusion XL C1E1DE52", "Generation time": "13.01191060245037", Comment: '{"prompt":"","steps":28,"height":1024,"width":1024,"scale":7,"uncond_scale":1,"cfg_rescale":0,"seed":1327009148,"n_samples":1,"hide_debug_overlay":false,"noise_schedule":"native","legacy_v3_extend":false,"reference_information_extracted_multiple":[],"reference_strength_multiple":[],"sampler":"k_euler_ancestral","controlnet_strength":1,"controlnet_model":null,"dynamic_thresholding":false,"dynamic_thresholding_percentile":0.999,"dynamic_thresholding_mimic_scale":10,"sm":true,"sm_dyn":true,"skip_cfg_below_sigma":0,"lora_unet_weights":null,"lora_clip_weights":null,"uc":"","request_type":"PromptGenerateRequest","signed_hash":"860jyd/93XmJMreXOqM59aYXMICqYxeEm7DfUHTHLZ6yHV1fRRngrKzCAkeKBT4x8CHHRj+aaBzTJb6PDY/Rhr=="}' };

const copy = (value) => {
  try {
    toClipboard(value);
    ElMessage({
      message: "复制成功",
      type: "success",
    });
  } catch (e) {
    console.log(e);
    ElMessage({
      message: "复制失败",
      type: "warning",
    });
  }
};

const showCopyBtn = (title) => {
  if (!title) return false
  if (
    title == "Description" ||
    title == "Comment" ||
    title == "完整生成信息" ||
    title.indexOf("提示词") != -1
  ) {
    return true;
  }
  return false;
};

const cleanData = () => {
  imgFileRef.value = null
  modelFileRef.value = null
  imgfileInfoRef.value = null
  modelFileInfoRef.value = null
  exifRef.value = null
  jsonData.value = null
}

async function handleUpload(file) {
  console.log(file);
  cleanData()

  let fileExt = file.name.split(".").pop().toLowerCase();
  if (availableImgExt.indexOf(fileExt) != -1) {
    imgFileRef.value = file;
    inspectImage(file)
  } else {
    ElMessage({
      message: "解析失败，该文件可能不是一个正常的图片/模型文件。",
      type: "error",
    });
  }
  return false;
}

const inspectImage = async (file) => {
  await readImageBase64()
  // exifRef.value = await readExif(file)
  imgfileInfoRef.value = await readFileInfo(file)
}

async function readFileInfo(file) {
  jsonData.value = null
  let parsed = []
  let metadataType = "None"

  let metadata = await getStealthExif(imageRef.value.src)
  if (metadata) {
    metadataType = "NovelAI"
    parsed = Object.keys(metadata).map((key) => {
      return {
        keyword: key,
        text: metadata[key],
      }
    });
  } else {
    metadataType = "None"
    ElMessage({
      message: "没有找到隐匿水印信息，这可能不是一张由NAI生成的图片，或者因压缩和修改而丢失了水印。",
      type: "warning",
    });
    return [{ key: "prompt", value: "" }, { key: "uc", value: "" }, { key: "Software", value: "NovelAI" }, { key: "Source", value: "Stable Diffusion XL C1E1DE52" }]
  }

  let ok = []
  const commentJson = JSON.parse(metadata["Comment"]);
  ok.push({ key: "prompt", value: commentJson.prompt });
  ok.push({ key: "uc", value: commentJson.uc });
  ok.push({ key: "Software", value: metadata["Software"] });
  ok.push({ key: "Source", value: metadata["Source"] });

  return ok
}

const readImageBase64 = async () => {
  imageRef.value = null;
  let result = await asyncFileReaderAsDataURL(imgFileRef.value)
  const image = new Image();
  image.src = result;
  await image.decode();
  const { width, height } = image;
  imageRef.value = {
    width,
    height,
    src: result,
  };
  imageMaxSizeRef.value = width;
}

const printableBytes = (size) => {
  const printable = (d, z) => {
    return `${d.toFixed(2)} ${z}`;
  };

  let kb = size / 1024;
  if (kb < 1024) {
    return printable(kb, "KB");
  }
  let mb = kb / 1024;
  if (mb < 1024) {
    return printable(mb, "MB");
  }

  let gb = mb / 1024;
  return printable(gb, "GB");
};

const saveMetadata = async () => {
  // 重新读取图片中的隐匿水印
  let existingMetadata = await getStealthExif(imageRef.value.src);

  // 从 imgfileInfoRef 中读取编辑后的数据
  const updatedMetadata = imgfileInfoRef.value.reduce((acc, item) => {
    acc[item.key] = item.value;
    return acc;
  }, {});

  // 更新隐匿水印中的信息
  if (!existingMetadata) {
    ElMessage({
      message: "使用模板元信息内容写入图片",
      type: "info",
    });
    existingMetadata = templateMetadata
  }

  if (existingMetadata) {
    // 更新 Comment 字段中的 prompt 和 uc
    if (existingMetadata["Comment"]) {
      const commentJson = JSON.parse(existingMetadata["Comment"]);
      commentJson.prompt = updatedMetadata["prompt"];
      commentJson.uc = updatedMetadata["uc"];

      // 生成请求签名
      const randomBytes = crypto.getRandomValues(new Uint8Array(64));
      const signature = btoa(String.fromCharCode.apply(null, randomBytes));
      commentJson.signed_hash = signature;

      // 更改尺寸信息
      const img = new Image();
      img.src = imageRef.value.src;
      await img.decode();
      commentJson["width"] = img.width;
      commentJson["height"] = img.height;

      // 更改种子信息
      const seed = Math.floor(Math.random() * 9000000000) + 1000000000;
      commentJson["seed"] = seed

      existingMetadata["Comment"] = JSON.stringify(commentJson);
    }

    // 更新水印中的 Description 字段
    existingMetadata["Description"] = updatedMetadata["prompt"];

    // 更改生成用时
    const generationTime = (Math.random() * 5 + 10);
    existingMetadata["Generation time"] = generationTime;

    // 更新 Software 和 Source 字段
    existingMetadata["Software"] = updatedMetadata["Software"];
    existingMetadata["Source"] = updatedMetadata["Source"];

    const imageDataUrl = imageRef.value.src;
    const outputImageDataUrl = await embedStealthExif(imageDataUrl, JSON.stringify(existingMetadata));

    // 替换页面中显示的图片
    imageRef.value.src = outputImageDataUrl;

    ElMessage({
      message: "元信息已保存并嵌入图片",
      type: "success",
    });
  } else {
    ElMessage({
      message: "无法获取水印信息，保存失败。",
      type: "error",
    });
  }
};

</script>
