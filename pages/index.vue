<script setup lang="ts">
const config = useRuntimeConfig();
useHead({
  title: "Mouse Select Image Translator",
});

const previewCanvas = ref<HTMLCanvasElement | null>(null);
const selectionBox = ref<HTMLDivElement | null>(null);
const selectionBoxStyle = ref({
  display: "none",
  left: "0px",
  top: "0px",
  width: "0px",
  height: "0px",
});
let isSelecting = false;
let startX = 0;
let startY = 0;

function handleImageUpload(file: File) {
  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      if (previewCanvas.value) {
        // Set the canvas size to match the real image size
        previewCanvas.value.width = img.naturalWidth;
        previewCanvas.value.height = img.naturalHeight;

        const ctx = previewCanvas.value.getContext("2d");
        ctx?.drawImage(img, 0, 0);
      }
    };
    img.src = e.target.result as string;
  };
  reader.readAsDataURL(file);
}

// Mouse event listeners for drawing the selection box
function startSelection(event: MouseEvent) {
  const rect = previewCanvas.value?.getBoundingClientRect();
  if (rect) {
    startX = event.clientX - rect.left;
    startY = event.clientY - rect.top;
    isSelecting = true;
    selectionBoxStyle.value.display = "block";
    selectionBoxStyle.value.left = `${startX}px`;
    selectionBoxStyle.value.top = `${startY}px`;
  }
}

function updateSelection(event: MouseEvent) {
  if (isSelecting && previewCanvas.value) {
    const rect = previewCanvas.value.getBoundingClientRect();
    const currentX = event.clientX - rect.left;
    const currentY = event.clientY - rect.top;
    const width = currentX - startX;
    const height = currentY - startY;
    selectionBoxStyle.value.width = `${Math.abs(width)}px`;
    selectionBoxStyle.value.height = `${Math.abs(height)}px`;
    selectionBoxStyle.value.left = `${width > 0 ? startX : startX + width}px`;
    selectionBoxStyle.value.top = `${height > 0 ? startY : startY + height}px`;
  }
}

function endSelection(event: MouseEvent) {
  if (isSelecting) {
    isSelecting = false;
  }
}

onMounted(() => {
  if (previewCanvas.value) {
    previewCanvas.value.addEventListener("mousedown", startSelection);
    previewCanvas.value.addEventListener("mousemove", updateSelection);
    window.addEventListener("mouseup", endSelection);
  }
});

onBeforeUnmount(() => {
  if (previewCanvas.value) {
    previewCanvas.value.removeEventListener("mousedown", startSelection);
    previewCanvas.value.removeEventListener("mousemove", updateSelection);
    window.removeEventListener("mouseup", endSelection);
  }
});

const languages = {
  CHS: "简体中文",
  CHT: "繁體中文",
  JPN: "日本語",
  ENG: "English",
  KOR: "한국어",
  VIN: "Tiếng Việt",
  CSY: "čeština",
  NLD: "Nederlands",
  FRA: "français",
  DEU: "Deutsch",
  HUN: "magyar nyelv",
  ITA: "italiano",
  PLK: "polski",
  PTB: "português",
  ROM: "limba română",
  RUS: "русский язык",
  ESP: "español",
  TRK: "Türk dili",
};
const language = useLocalStorage("trans_language", "CHS");

const sizes = {
  S: "1024px",
  M: "1536px",
  L: "2048px",
  X: "2560px",
};
const size = ref("L");

const detectors = {
  default: "Default",
  ctd: "Comic Text Detector",
};
const detector = ref("default");

const directions = {
  default: "Follow language",
  auto: "Follow image",
  h: "All horizontal",
  v: "All vertical",
};
const direction = useSessionStorage("trans_direction", "default");

const translators = {
  none: "Remove text",
  "gpt3.5": "GPT-3.5",
  youdao: "Youdao",
  baidu: "Baidu",
  google: "Google",
  deepl: "DeepL",
  papago: "Papago",
  offline: "Sugoi / M2M100",
  original: "Untranslated",
};
const translator = useSessionStorage("trans_translator", "google");

const acceptTypes = ["image/png", "image/jpeg", "image/bmp", "image/webp"];
const file = shallowRef<File | null>(null);

function onDrop(e: DragEvent) {
  const f = e.dataTransfer?.files[0];
  if (f && acceptTypes.includes(f.type)) file.value = f;
}
function onFileChange(e: Event) {
  const f = (e.target as HTMLInputElement).files?.[0];
  if (f && acceptTypes.includes(f.type)) file.value = f;
}
useEventListener(document, "paste", (e: ClipboardEvent) => {
  const f = e.clipboardData?.files[0];
  if (f && acceptTypes.includes(f.type)) file.value = f;
});

const fileUri = ref("");
watch(file, (file) => {
  if (fileUri.value) URL.revokeObjectURL(fileUri.value);
  fileUri.value = file ? URL.createObjectURL(file) : "";
});

interface TaskResult {
  translation_mask: string;
}

interface QueryV1MessagePending {
  type: "pending";
  pos: number;
}
interface QueryV1MessageStatus {
  type: "status";
  status: string;
}
interface QueryV1MessageResult {
  type: "result";
  result: TaskResult;
}
interface QueryV1MessageError {
  type: "error";
  error_id?: string | null;
}
interface QueryV1MessageNotFound {
  type: "not_found";
}
type QueryV1Message =
  | QueryV1MessagePending
  | QueryV1MessageStatus
  | QueryV1MessageResult
  | QueryV1MessageError
  | QueryV1MessageNotFound;

const taskId = ref("");
const errorId = ref("");
const errorStatus = ref("");
const resultBlob = ref<Blob | null>(null);
const status = ref("");

async function upload() {
  if (!file.value) return;

  errorId.value = "";
  errorStatus.value = "";
  resultBlob.value = null;
  status.value = "uploading";

  const formData = new FormData();

  if (previewCanvas.value) {
    const rect = previewCanvas.value.getBoundingClientRect();
    const scaleX = previewCanvas.value.width / rect.width;
    const scaleY = previewCanvas.value.height / rect.height;
    const x = parseInt(selectionBoxStyle.value.left) * scaleX;
    const y = parseInt(selectionBoxStyle.value.top) * scaleY;
    const width = parseInt(selectionBoxStyle.value.width) * scaleX;
    const height = parseInt(selectionBoxStyle.value.height) * scaleY;
    // Extract the selected area from the original image

    // Create a new canvas for the selected area
    const selectedCanvas = document.createElement("canvas");
    selectedCanvas.width = width;
    selectedCanvas.height = height;
    const selectedCtx = selectedCanvas.getContext("2d");

    // Draw the selected area of the original image onto the new canvas
    selectedCtx.drawImage(
      previewCanvas.value,
      x,
      y,
      width,
      height, // Source rectangle (selected area)
      0,
      0,
      width,
      height // Destination rectangle (same size as source)
    );

    // Wait for the blob to be created from the canvas
    const blob = await new Promise<Blob | null>((resolve) => {
      selectedCanvas.toBlob(resolve, file.value.type);
    });

    // If the blob is not null, append it to the formData
    if (blob) {
      formData.append("file", blob);
    } else {
      // Handle the error case where the blob could not be created
      errorStatus.value = "Failed to create image blob.";
      return;
    }
  } else {
    // Handle the error case where the canvas is not available
    errorStatus.value = "Canvas is not available.";
    return;
  }

  formData.append("mime", file.value.type);
  formData.append("target_language", language.value);
  formData.append("detector", detector.value);
  formData.append("direction", direction.value);
  formData.append("translator", translator.value);
  formData.append("size", size.value);

  const res = await $fetch<{
    id: string;
    status: string;
    result?: TaskResult | null;
  }>(`${config.public.apiBase}/task/upload/v1`, {
    method: "PUT",
    body: formData,
    onResponseError({ response }) {
      errorId.value = response._data;
      errorStatus.value = `${response.status} ${response.statusText}\n${response._data}`;
    },
    onRequestError({ error }) {
      errorStatus.value = error.toString();
    },
  });

  status.value = "pending";
  taskId.value = res.id;

  let result: TaskResult;
  if (res.result) {
    result = res.result;
  } else {
    const socket = new WebSocket(`${config.public.wsBase}/task/${taskId.value}/event/v1`);
    result = await new Promise<TaskResult>((resolve, reject) => {
      socket.addEventListener("message", (e) => {
        try {
          const data: QueryV1Message = JSON.parse(e.data);
          if (data.type === "pending") {
            status.value = `pending (${data.pos} in queue)`;
          } else if (data.type === "status") {
            status.value = data.status;
          } else if (data.type === "result") {
            socket.close();
            resolve(data.result);
          } else if (data.type === "error") {
            errorId.value = data.error_id ?? "undefined";
            socket.close();
            reject(new Error("error"));
          } else if (data.type === "not_found") {
            errorId.value = "not_found";
            socket.close();
            reject(new Error("not_found"));
          }
        } catch (e) {
          console.error(e);
        }
      });
    });
    document.getElementById("imguri").innerHTML =
      document.getElementById("imguri").innerHTML +
      `<img src="${result.translation_mask}" style="position: absolute; top: ${startY}px; left: ${startX}px; width: auto;"/>`;
    status.value = "";
    socket.close();
  }

  // status.value = "rendering";

  // // layer translation_mask on top of original image
  // const canvas = document.createElement("canvas");
  // const canvasCtx = canvas.getContext("2d")!;
  // // draw original image
  // const img = new Image();
  // img.src = fileUri.value;
  // await new Promise((resolve) => {
  //   img.onload = () => {
  //     canvas.width = img.width;
  //     canvas.height = img.height;
  //     canvasCtx.drawImage(img, 0, 0);
  //     resolve(null);
  //   };
  // });
  // // draw translation_mask
  // const img2 = new Image();
  // img2.src = result.translation_mask;
  // img2.crossOrigin = "anonymous";
  // await new Promise((resolve) => {
  //   img2.onload = () => {
  //     canvasCtx.drawImage(img2, 0, 0);
  //     resolve(null);
  //   };
  // });

  // canvas.toBlob((blob) => {
  //   resultBlob.value = blob;
  // }, "image/png");
}

// watch(resultBlob, (blob) => {
//   if (resultUri.value) URL.revokeObjectURL(resultUri.value);
//   resultUri.value = blob ? URL.createObjectURL(blob) : "";
// });

// function saveAsPNG() {
//   if (!resultUri.value || !taskId.value) return;

//   const a = document.createElement("a");
//   a.href = resultUri.value;
//   a.download = `translation-${taskId}.png`;
//   a.classList.add("hidden");
//   document.body.appendChild(a);
//   a.click();
//   document.body.removeChild(a);
// }

function reset() {
  file.value = null;
  taskId.value = "";
  errorId.value = "";
  errorStatus.value = "";
  resultBlob.value = null;
  status.value = "";
}

watch(file, (newFile) => {
  if (newFile) {
    handleImageUpload(newFile);
  }
});
</script>

<template>
  <div class="flex-1 flex justify-between items-center gap-6">
    <div class="flex-1 flex flex-col min-w-0 h-full">
      <div
        v-if="fileUri"
        class="relative"
        @mousedown="startSelection"
        @mousemove="updateSelection"
        @mouseup="endSelection"
        @mouseleave="endSelection"
      >
        <div id="imguri"></div>
        <canvas ref="previewCanvas" class="width: 1024;"></canvas>
        <div
          ref="selectionBox"
          class="absolute border-2 border-dashed border-black pointer-events-none"
          :style="selectionBoxStyle"
        ></div>
      </div>
      <label
        class="flex-1 rounded-2xl"
        @dragenter.prevent
        @dragover.prevent
        @dragleave.prevent
        @drop.prevent="onDrop"
      >
        <input
          type="file"
          :accept="acceptTypes.join(',')"
          class="hidden"
          @change="onFileChange"
        />

        <div class="grid place-items-center w-full h-full rounded-2xl text-zinc-800">
          <div v-if="errorId || errorStatus" class="flex flex-col items-center gap-4">
            <div class="flex justify-center items-center gap-1 text-lg">
              <div class="i-ri:close-line w-5 h-5" />
              Oops!
            </div>

            <div class="flex flex-col items-center">
              We experienced an issue while translating your image.
              <div v-if="errorStatus" class="text-sm">{{ errorStatus }}</div>
              <div v-if="taskId" class="text-sm font-mono">Task ID: {{ taskId }}</div>
              <div v-if="errorId" class="text-sm font-mono">Error ID: {{ errorId }}</div>
            </div>

            <div>
              <button
                class="px-6 py-1 rounded-full text-fuchsia-600 border-2 border-fuchsia-300"
                @click.prevent="reset"
              >
                Try another one
              </button>
            </div>
          </div>

          <div v-else-if="status" class="flex flex-col items-center gap-4">
            <div class="flex justify-center items-center gap-1 text-lg">
              <div class="i-ri:loader-4-line w-5 h-5 animate-spin" />
              Translating...
            </div>

            <div class="text-zinc-700" style="height: 1000px;">
              {{ status }}
            </div>
          </div>

          <div v-else-if="fileUri" class="flex flex-col sm:flex-row items-center gap-8">
            <div class="flex flex-col gap-2 pointer-events-none">
              <div class="flex items-center gap-1 text-lg">
                <div class="i-ri:settings-2-line w-5 h-5" />
                Options
              </div>

              <label class="pointer-events-auto">
                <span class="ml-2 text-sm">Language</span>
                <UListbox v-model="language" class="w-56" :items="languages" />
              </label>
              <label class="pointer-events-auto">
                <span class="ml-2 text-sm">Detection Resolution</span>
                <UListbox v-model="size" class="w-56" :items="sizes" />
              </label>
              <label class="pointer-events-auto">
                <span class="ml-2 text-sm">Text Detector</span>
                <UListbox v-model="detector" class="w-56" :items="detectors" />
              </label>
              <label class="pointer-events-auto">
                <span class="ml-2 text-sm">Text Direction</span>
                <UListbox v-model="direction" class="w-56" :items="directions" />
              </label>
              <label class="pointer-events-auto">
                <span class="ml-2 text-sm">Translator</span>
                <UListbox v-model="translator" class="w-56" :items="translators" />
                <div class="mt-1 ml-1 w-54 text-xs">
                  If Google can't process your image, try using other translators!
                </div>
              </label>

              <button
                class="py-1 w-56 text-center rounded-full text-fuchsia-600 border border-fuchsia-300 pointer-events-auto"
                @click.prevent="upload"
              >
                Translate
              </button>
            </div>
          </div>

          <div v-else class="flex flex-col items-center justify-center gap-10vh">
            <div>&nbsp;</div>

            <div class="flex items-center gap-2 max-w-80vw cursor-pointer">
              <i class="i-ri:image-add-line w-8 h-8 text-zinc-500" />
              Paste an image, click to select, or drag and drop
            </div>
          </div>
        </div>
      </label>
    </div>
  </div>
</template>
