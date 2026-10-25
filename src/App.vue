<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, toRef, watch } from "vue";

import CollapsibleSection from "./components/CollapsibleSection.vue";
import ModelSelector from "./components/ModelSelector.vue";
import SearchableSelect from "./components/SearchableSelect.vue";
import ImageDropzone from "./components/ImageDropzone.vue";
import MediaDropzone from "./components/MediaDropzone.vue";
import StatusChip from "./components/StatusChip.vue";
import { cancelJob, getCapabilities, getJob, submitImageJob, submitVideoJob } from "./lib/api";
import { applyRecipeDefaults, buildRequestBodyForMode, createBlankForm, formFromCapabilities } from "./lib/form";
import { availableCacheModes, featuresForMode, supportsEta, supportsFlowShift, usesSCM } from "./lib/generation-features";
import { IMAGE_INPUTS, VIDEO_IMAGE_INPUTS } from "./lib/image-inputs";
import { assignImageEntries, clearImageEntries, filesToImageEntries, removeImageEntry } from "./lib/images";
import { createStoredRef, normalizePollIntervalMs, normalizePreviewInterval } from "./lib/settings";
import { useTheme } from "./lib/theme";
import { useMediaInput } from "./lib/media-input";
import type {
    Capabilities,
    ModelStatus,
    Recipe,
    GenerationForm,
    GenerationFeatures,
    GenerationMode,
    ImageEntry,
    ImageTarget,
    Job,
    SampleParams,
} from "./lib/types";

const baseUrl = createStoredRef<string>("sdcpp-webui-base-url", "", (value: unknown) => String(value || ""));
const pollIntervalMs = createStoredRef<number>("sdcpp-webui-poll-interval-ms", 100, normalizePollIntervalMs);
const preferredPreviewMode = createStoredRef<string>("sdcpp-webui-preview-mode", "none", (value: unknown) => typeof value === "string" ? value : "none");
const previewInterval = createStoredRef<number>("sdcpp-webui-preview-interval", 1, normalizePreviewInterval);
const activeTab = ref<"image" | "video" | "settings">("image");
const theme = useTheme();
const selectedGenerationTab = ref<"image" | "video">("image");
const generationMode = computed<GenerationMode>(() => selectedGenerationTab.value === "video" ? "video" : "image");
const lightboxOpen = ref(false);
const lightboxImageSrc = ref("");
const lightboxImageAlt = ref("");
const sectionState = reactive<Record<string, boolean>>({
    sample: true,
    sampleAdvanced: false,
    guidance: true,
    guidanceAdvanced: false,
    highNoise: false,
    highNoiseSample: true,
    highNoiseGuidance: true,
    conditioning: false,
    auxiliaryImages: false,
    referenceMedia: true,
    lora: false,
    vaeTiling: false,
    cache: false,
});
const loadingCapabilities = ref(false);
const capabilitiesError = ref("");
const serviceOnline = ref(false);
const capabilities = ref<Capabilities | null>(null);
const modelStatus = ref<ModelStatus | null>(null);
const selectedRecipe = ref<Recipe | null>(null);
const applyRecipeDefaultsOnSelect = createStoredRef<boolean>("sdcpp-webui-apply-recipe-defaults", true, (value: unknown) => typeof value === "boolean" ? value : true);
const modelSelection = ref<Record<string, string>>({});
const modelReady = computed(() => modelStatus.value ? modelStatus.value.state === "ready" : Boolean(capabilities.value?.model?.name));
const currentJob = ref<Job | null>(null);
const selectedOutputIndex = ref(0);
const statusMessage = ref("");
const statusTone = ref("");
const cancelling = ref(false);
let pollVersion = 0;
const form = reactive<GenerationForm>(createBlankForm());
const videoInput = useMediaInput("video", toRef(form, "ref_video"));
const audioInput = useMediaInput("audio", toRef(form, "ref_audio"));
const submitting = ref(false);

let pollTimer = 0;
let elapsedTimer = 0;
const nowMilliseconds = ref(Date.now());

const modelName = computed(() => {
    const model = capabilities.value?.model;
    return model?.stem || model?.name || "No model loaded";
});

const supportedModes = computed(() => capabilities.value?.supported_modes || []);
const loadedRecipeId = computed(() => modelStatus.value?.runtime_recipe?.id || modelStatus.value?.recipe);
const recipeMode = computed<GenerationMode>(() => selectedRecipe.value?.source === "command_line"
    ? capabilities.value?.current_mode === "vid_gen" ? "video" : "image"
    : selectedRecipe.value?.defaults?.video_frames !== undefined ? "video" : "image");
const previewingRecipe = computed(() => Boolean(selectedRecipe.value) &&
    (!modelReady.value || loadedRecipeId.value !== selectedRecipe.value?.id));
const supportsImageMode = computed(() => {
    if (previewingRecipe.value) return recipeMode.value === "image";
    return modelReady.value && (!supportedModes.value.length || supportedModes.value.includes("img_gen"));
});
const supportsVideoMode = computed(() => {
    if (previewingRecipe.value) return recipeMode.value === "video";
    return modelReady.value && (!supportedModes.value.length || supportedModes.value.includes("vid_gen"));
});
const selectedModeKey = computed<"img_gen" | "vid_gen">(() => generationMode.value === "video" ? "vid_gen" : "img_gen");
const canGenerate = computed(() => modelReady.value && !previewingRecipe.value &&
    (!supportedModes.value.length || supportedModes.value.includes(selectedModeKey.value)));
const currentJobModeKey = computed<"img_gen" | "vid_gen">(() => currentJob.value?.kind || selectedModeKey.value);
const selectedModeFeatures = computed(() => {
    return capabilities.value?.features_by_mode?.[selectedModeKey.value] || {};
});
const currentJobFeatures = computed(() => {
    return capabilities.value?.features_by_mode?.[currentJobModeKey.value] || selectedModeFeatures.value;
});
const imageOutputFormats = computed(() => {
    return capabilities.value?.output_formats_by_mode?.img_gen || ["png", "jpeg"];
});
const videoOutputFormats = computed(() => {
    return capabilities.value?.output_formats_by_mode?.vid_gen || ["webm", "avi"];
});
const outputFormats = computed(() => generationMode.value === "video" ? videoOutputFormats.value : imageOutputFormats.value);
const samplers = computed(() => capabilities.value?.samplers || ["default"]);
const schedulers = computed(() => capabilities.value?.schedulers || ["default"]);
const availableLoras = computed(() => capabilities.value?.loras || []);
const generationFeatures = computed<GenerationFeatures>(() => {
    const loadedSelection = !previewingRecipe.value && loadedRecipeId.value === selectedRecipe.value?.id;
    const declared = loadedSelection ? capabilities.value?.generation_features : selectedRecipe.value?.effective_features;
    const features = { ...(declared || capabilities.value?.generation_features || {}) };
    if (selectedRecipe.value && selectedRecipe.value.source !== "command_line") {
        for (const [feature, slots] of Object.entries({ high_noise: ["high_noise_diffusion_model"], preview_tae: ["tae", "taesd"] })) {
            features[feature] = Boolean(features[feature] && slots.some((slot) => modelSelection.value[slot]));
        }
        if (modelSelection.value.pulid_weights) features.slg = false;
    }
    return featuresForMode(features, generationMode.value);
});
const hasFeature = (name: string): boolean => Boolean(generationFeatures.value[name]);
const mediaProcessing = computed(() => (hasFeature("ref_videos") && form.ref_video?.processing) ||
    (hasFeature("ref_audios") && form.ref_audio?.processing));
const mediaSummary = computed(() => [hasFeature("ref_videos") && form.ref_video ? "1 video" : "",
    hasFeature("ref_audios") && form.ref_audio ? "1 audio" : ""].filter(Boolean).join(" · ") || "No media");
const sampleHasEta = (sample: SampleParams): boolean => supportsEta(sample.sample_method, generationFeatures.value);
const sampleHasFlowShift = (sample: SampleParams): boolean => supportsFlowShift(sample.scheduler, generationFeatures.value);
const sampleHasExtras = (sample: SampleParams): boolean => sampleHasEta(sample) || hasFeature("shifted_timestep");
const hasGuidanceExtras = computed(() => hasFeature("img_cfg") || hasFeature("slg"));
const hasConditioning = computed(() => ["clip_skip", "controlnet", "vace"].some(hasFeature));
const cacheModes = computed(() => availableCacheModes(generationFeatures.value));
const currentImageInputs = computed(() => (generationMode.value === "video" ? VIDEO_IMAGE_INPUTS : IMAGE_INPUTS).filter((input) => hasFeature(input.target)));
const gridImageInputs = computed(() => currentImageInputs.value.filter((input) => input.layout === "grid"));
const fullImageInputs = computed(() => currentImageInputs.value.filter((input) => input.layout === "full"));
const queueLimit = computed(() => capabilities.value?.limits?.max_queue_size ?? "unknown");
const canCancelQueued = computed(() => Boolean(currentJobFeatures.value.cancel_queued));
const canCancelGenerating = computed(() => Boolean(currentJobFeatures.value.cancel_generating));
const supportsPreview = computed(() => hasFeature("preview"));
const previewModes = computed(() => capabilities.value?.preview_modes || ["none", "proj", "tae", "vae"]);
const previewMode = computed(() => supportsPreview.value && previewModes.value.includes(preferredPreviewMode.value) &&
    (preferredPreviewMode.value !== "tae" || hasFeature("preview_tae")) ? preferredPreviewMode.value : "none");

const currentStatus = computed(() => currentJob.value?.status || "idle");
const currentJobKind = computed(() => currentJob.value?.kind || null);
const currentImages = computed(() => currentJobKind.value === "img_gen" ? currentJob.value?.result?.images || [] : []);
const selectedImage = computed(() => {
    if (!currentImages.value.length) {
        return null;
    }
    const index = Math.min(selectedOutputIndex.value, currentImages.value.length - 1);
    const image = currentImages.value[index];
    const format = currentJob.value?.result?.output_format || form.output_format || "png";
    return `data:image/${format};base64,${image.b64_json}`;
});
const videoMimeType = computed(() => currentJobKind.value === "vid_gen" ? currentJob.value?.result?.mime_type || "" : "");
const videoFrameCount = computed(() => currentJobKind.value === "vid_gen" ? currentJob.value?.result?.frame_count || 0 : 0);
const videoFps = computed(() => currentJobKind.value === "vid_gen" ? currentJob.value?.result?.fps || 0 : 0);
const videoPreviewSrc = computed(() => {
    if (currentJobKind.value !== "vid_gen" || !currentJob.value?.result?.b64_json) {
        return null;
    }
    if (currentJob.value?.result?.output_format === "avi") {
        return null;
    }
    if (!videoMimeType.value.startsWith("video/")) {
        return null;
    }
    return `data:${videoMimeType.value};base64,${currentJob.value.result.b64_json}`;
});
const animatedVideoImageSrc = computed(() => {
    if (currentJobKind.value !== "vid_gen" || !currentJob.value?.result?.b64_json) {
        return null;
    }
    if (videoMimeType.value !== "image/webp") {
        return null;
    }
    return `data:image/webp;base64,${currentJob.value.result.b64_json}`;
});
const previewImageSrc = computed(() => animatedVideoImageSrc.value || selectedImage.value);
const livePreviewSrc = computed(() => {
    if (currentStatus.value !== "generating" || !currentJob.value?.preview?.b64_json) {
        return null;
    }
    return `data:image/png;base64,${currentJob.value.preview.b64_json}`;
});
const livePreviewPass = computed(() => currentJob.value?.preview?.pass ?? 1);
const livePreviewStep = computed(() => currentJob.value?.preview?.step ?? 0);
const livePreviewTotalSteps = computed(() => currentJob.value?.preview?.total_steps ?? 0);
const downloadableSrc = computed(() => {
    const result = currentJob.value?.result;
    if (currentJobKind.value === "vid_gen" && result?.b64_json && videoMimeType.value) {
        return `data:${videoMimeType.value};base64,${result.b64_json}`;
    }
    return selectedImage.value;
});

const canCancelCurrentJob = computed(() => {
    if (!currentJob.value || cancelling.value || currentJob.value.cancel_requested) {
        return false;
    }
    if (currentStatus.value === "queued") {
        return canCancelQueued.value;
    }
    if (currentStatus.value === "generating") {
        return canCancelGenerating.value;
    }
    return false;
});

const isJobRunning = computed(() => {
    return currentStatus.value === "queued" || currentStatus.value === "generating";
});

function buildSampleSummary(sample: SampleParams): string {
    const scheduler = sample.scheduler || "default";
    const method = sample.sample_method || "default";
    const steps = sample.sample_steps || 0;
    const flowShift = sample.flow_shift === "" || sample.flow_shift == null
        ? "flow auto"
        : `flow ${sample.flow_shift}`;
    return [scheduler, ...(sampleHasFlowShift(sample) ? [flowShift] : []), method, `${steps} steps`].join(" · ");
}

function buildGuidanceSummary(sample: SampleParams): string {
    const cfg = sample.guidance.txt_cfg;
    const distilled = sample.guidance.distilled_guidance;
    return `cfg ${cfg}` + (hasFeature("distilled_guidance") ? ` · distilled ${distilled}` : "");
}

const sampleSummary = computed(() => buildSampleSummary(form.sample_params));
const guidanceSummary = computed(() => buildGuidanceSummary(form.sample_params));
const highNoiseSummary = computed(() => {
    return `moe ${formatSummaryNumber(form.moe_boundary)} · ${buildSampleSummary(form.high_noise_sample_params)} · ${buildGuidanceSummary(form.high_noise_sample_params)}`;
});
const loraSummary = computed(() => {
    const count = form.lora.filter((item) => item.path.trim()).length;
    if (!count) {
        return "No LoRA";
    }
    return `${count} configured`;
});
const imageInputsSummary = computed(() => {
    const parts: string[] = [];
    if (hasFeature("init_image") && form.init_image) parts.push(generationMode.value === "video" ? "start" : "init");
    if (generationMode.value === "image") {
        if (hasFeature("mask_image") && form.mask_image) parts.push("mask");
        if (hasFeature("control_image") && form.control_image) parts.push("control");
        if (hasFeature("ip_adapter") && form.ip_adapter_image) parts.push("IP-Adapter");
        if (hasFeature("ref_images") && form.ref_images.length) parts.push(`${form.ref_images.length} refs`);
    } else {
        if (hasFeature("end_image") && form.end_image) parts.push("end");
        if (hasFeature("ref_images") && form.ref_images.length) parts.push(`${form.ref_images.length} refs`);
        if (hasFeature("control_frames") && form.control_frames.length) parts.push(`${form.control_frames.length} frames`);
    }
    return parts.length ? parts.join(" · ") : "No images";
});
const vaeTilingSummary = computed(() => {
    if (!form.vae_tiling_params.enabled) {
        return "Disabled";
    }
    return `${form.vae_tiling_params.tile_size_w}×${form.vae_tiling_params.tile_size_h} · overlap ${form.vae_tiling_params.target_overlap}`;
});
const cacheSummary = computed(() => {
    const mode = form.cache.mode || "disabled";
    if (mode === "disabled") {
        return "Disabled";
    }
    const option = String(form.cache.option || "").trim();
    return option ? `${mode} · ${option}` : mode;
});
const conditioningSummary = computed(() => {
    const parts: string[] = [];
    if (hasFeature("clip_skip")) parts.push(`clip_skip ${formatSummaryNumber(form.clip_skip, 0)}`);
    if (hasFeature("controlnet")) parts.push(`control ${formatSummaryNumber(form.control_strength)}`);
    if (hasFeature("vace")) parts.push(`vace ${formatSummaryNumber(form.vace_strength)}`);
    return parts.join(" · ");
});

function defaultOutputFormatForMode(mode: GenerationMode): string {
    if (mode === "video") {
        return videoOutputFormats.value[0] || "webm";
    }
    return imageOutputFormats.value[0] || "png";
}

function ensureOutputFormatForMode(mode = generationMode.value): void {
    const validFormats = mode === "video" ? videoOutputFormats.value : imageOutputFormats.value;
    if (!validFormats.length) {
        return;
    }
    if (!validFormats.includes(form.output_format)) {
        form.output_format = defaultOutputFormatForMode(mode);
    }
}

function setMessage(message: string, tone = ""): void {
    statusMessage.value = message;
    statusTone.value = tone;
}

function randomizeSeed(): void {
    form.seed = crypto.getRandomValues(new Uint32Array(1))[0];
}

function clearMessage(): void {
    setMessage("", "");
}

function deepAssign(target: Record<string, any>, ...sources: Record<string, any>[]): Record<string, any> {
    for (const source of sources) {
        if (!source) continue;
        for (const key of Object.keys(source)) {
            const sv = source[key];
            if (sv !== null && typeof sv === "object" && !Array.isArray(sv) &&
                target[key] !== null && typeof target[key] === "object" && !Array.isArray(target[key])) {
                deepAssign(target[key], sv);
            } else {
                target[key] = sv;
            }
        }
    }
    return target;
}

function applyForm(nextForm: GenerationForm): void {
    deepAssign(form, createBlankForm(), nextForm);
}

function updateCatalogRecipe(recipe: Recipe | null): void {
    if (selectedRecipe.value) selectedRecipe.value = recipe;
    else if (recipe && (recipe.source === "command_line" || (!modelReady.value && modelStatus.value?.state !== "loading"))) selectRecipe(recipe);
}

function selectRecipe(recipe: Recipe): void {
    selectedRecipe.value = recipe;
    if (applyRecipeDefaultsOnSelect.value) {
        if (recipe.source === "command_line" && capabilities.value?.runtime?.origin === "command_line") {
            const defaults = formFromCapabilities(capabilities.value);
            for (const key of ["width", "height", "video_frames", "fps", "strength"] as const) form[key] = defaults[key];
            deepAssign(form.sample_params, defaults.sample_params);
            deepAssign(form.high_noise_sample_params, defaults.high_noise_sample_params);
        } else {
            applyRecipeDefaults(form, recipe.defaults);
        }
    }
    selectedGenerationTab.value = recipeMode.value;
    activeTab.value = recipeMode.value;
    ensureOutputFormatForMode(recipeMode.value);
}

async function refreshCapabilities(): Promise<void> {
    loadingCapabilities.value = true;
    capabilitiesError.value = "";
    try {
        const response = await getCapabilities(baseUrl.value);
        const initializeForm = !capabilities.value;
        const enteringCommandLine = response.runtime?.origin === "command_line" && capabilities.value?.runtime?.origin !== "command_line";
        capabilities.value = response;
        if (response.runtime) modelStatus.value = response.runtime;
        serviceOnline.value = true;
        if (initializeForm) {
            const defaults = formFromCapabilities(response);
            defaults.prompt = form.prompt || defaults.prompt;
            defaults.negative_prompt = form.negative_prompt || defaults.negative_prompt;
            applyForm(defaults);
            if (selectedRecipe.value && applyRecipeDefaultsOnSelect.value) applyRecipeDefaults(form, selectedRecipe.value.defaults);
        }
        if (response.runtime?.runtime_recipe && (enteringCommandLine || !selectedRecipe.value)) {
            selectRecipe(response.runtime.runtime_recipe);
        }
        if (selectedGenerationTab.value === "image" && !supportsImageMode.value && supportsVideoMode.value) {
            selectedGenerationTab.value = "video";
        } else if (selectedGenerationTab.value === "video" && !supportsVideoMode.value && supportsImageMode.value) {
            selectedGenerationTab.value = "image";
        }
        if (activeTab.value === "image" && !supportsImageMode.value && supportsVideoMode.value) {
            activeTab.value = "video";
        } else if (activeTab.value === "video" && !supportsVideoMode.value && supportsImageMode.value) {
            activeTab.value = "image";
        }
        ensureOutputFormatForMode();
        clearMessage();
    } catch (error) {
        capabilitiesError.value = error instanceof Error ? error.message : String(error);
        serviceOnline.value = false;
        setMessage(capabilitiesError.value, "error");
    } finally {
        loadingCapabilities.value = false;
    }
}

function toggleSection(section: string): void {
    sectionState[section] = !sectionState[section];
}

function selectGenerationMode(mode: GenerationMode): void {
    if (mode === "image" && !supportsImageMode.value) {
        setMessage("Current model only supports video generation.", "error");
        return;
    }
    if (mode === "video" && !supportsVideoMode.value) {
        setMessage("Current model only supports image generation.", "error");
        return;
    }
    selectedGenerationTab.value = mode;
    activeTab.value = mode;
    ensureOutputFormatForMode(mode);
}

function stopPolling(): void {
    if (pollTimer) {
        window.clearTimeout(pollTimer);
        pollTimer = 0;
    }
}

function stopElapsedTimer(): void {
    if (elapsedTimer) {
        window.clearInterval(elapsedTimer);
        elapsedTimer = 0;
    }
}

function startElapsedTimer(): void {
    stopElapsedTimer();
    nowMilliseconds.value = Date.now();
    elapsedTimer = window.setInterval(() => {
        nowMilliseconds.value = Date.now();
    }, 100);
}

async function pollJob(id: string): Promise<void> {
    stopPolling();
    const version = ++pollVersion;
    try {
        const job = await getJob(baseUrl.value, id);
        if (version !== pollVersion) return;
        currentJob.value = job;
        serviceOnline.value = true;
        if (currentStatus.value === "queued" || currentStatus.value === "generating") {
            pollTimer = window.setTimeout(() => pollJob(id), normalizePollIntervalMs(pollIntervalMs.value));
            if (job.cancel_requested) {
                setMessage("Cancellation requested. Waiting for generation to stop.", "");
            } else {
                clearMessage();
            }
            return;
        }
        stopElapsedTimer();
        if (currentStatus.value === "completed") {
            setMessage(currentJob.value?.kind === "vid_gen" ? "Video generation completed." : "Image generation completed.", "success");
            return;
        }
        if (currentStatus.value === "cancelled") {
            setMessage("Job cancelled.", "error");
            return;
        }
        if (currentStatus.value === "failed") {
            setMessage(currentJob.value?.error?.message || "Generation failed.", "error");
        }
    } catch (error) {
        if (version !== pollVersion) return;
        serviceOnline.value = false;
        setMessage(error instanceof Error ? error.message : String(error), "error");
        if (currentJob.value?.id === id && isJobRunning.value) {
            pollTimer = window.setTimeout(() => pollJob(id), normalizePollIntervalMs(pollIntervalMs.value));
        } else {
            stopElapsedTimer();
        }
    }
}

async function generate(): Promise<void> {
    if (!canGenerate.value || isJobRunning.value || mediaProcessing.value || submitting.value) return;
    submitting.value = true;
    try {
        const request = buildRequestBodyForMode(generationMode.value, {
            ...form,
            preview_mode: previewMode.value,
            preview_interval: normalizePreviewInterval(previewInterval.value),
        }, generationFeatures.value);
        clearMessage();
        selectedOutputIndex.value = 0;
        startElapsedTimer();
        currentJob.value = generationMode.value === "video"
            ? await submitVideoJob(baseUrl.value, request)
            : await submitImageJob(baseUrl.value, request);
        await pollJob(currentJob.value.id);
    } catch (error) {
        stopElapsedTimer();
        setMessage(error instanceof Error ? error.message : String(error), "error");
    } finally {
        submitting.value = false;
    }
}

async function cancelCurrentJob(): Promise<void> {
    if (!currentJob.value?.id || !canCancelCurrentJob.value) {
        return;
    }
    const id = currentJob.value.id;
    cancelling.value = true;
    stopPolling();
    ++pollVersion;
    try {
        currentJob.value = await cancelJob(baseUrl.value, id);
        await pollJob(id);
    } catch (error) {
        setMessage(error instanceof Error ? error.message : String(error), "error");
        pollTimer = window.setTimeout(() => pollJob(id), normalizePollIntervalMs(pollIntervalMs.value));
    } finally {
        cancelling.value = false;
    }
}

function addLora(): void {
    form.lora.push({
        path: "",
        multiplier: 1,
        is_high_noise: false,
    });
}

function removeLora(index: number): void {
    form.lora.splice(index, 1);
}

async function assignImages(target: ImageTarget, files: FileList): Promise<void> {
    const images = await filesToImageEntries(files);
    if (!images.length) {
        return;
    }
    assignImageEntries(form, target, images);
}

function clearImage(target: ImageTarget): void {
    clearImageEntries(form, target);
}

function getFormImage(target: ImageTarget): ImageEntry | null {
    if (target === "ref_images" || target === "control_frames") return null;
    return form[target];
}

function openImageEntry(image: ImageEntry | null): void {
    openLightbox(image?.dataUrl, image?.name);
}

function removeCollectionImage(target: "ref_images" | "control_frames", index: number): void {
    removeImageEntry(form, target, index);
}

function selectOutput(index: number): void {
    selectedOutputIndex.value = index;
}

function formatUnixTime(milliseconds: number | undefined): string {
    if (!milliseconds) {
        return "No job";
    }
    return new Date(milliseconds).toLocaleString();
}

function formatElapsed(job: Job | null): string {
    if (!job?.started) return "Idle";
    const elapsed = ((job.completed || nowMilliseconds.value) - job.started) / 1000;
    const total = Math.round(Math.max(0, elapsed) * 100) / 100;
    if (total < 60) return `${total.toFixed(2)}s`;
    const minutes = Math.floor(total / 60);
    return `${minutes}m ${(total - minutes * 60).toFixed(2)}s`;
}

function formatSummaryNumber(value: number, digits = 3): string {
    const numeric = Number(value ?? 0);
    if (!Number.isFinite(numeric)) {
        return "0";
    }
    return Number(numeric.toFixed(digits)).toString();
}

function downloadSelected(): void {
    if (!downloadableSrc.value) {
        return;
    }
    const link = document.createElement("a");
    link.href = downloadableSrc.value;
    link.download = `${currentJob.value?.id || "output"}.${currentJob.value?.result?.output_format || form.output_format}`;
    document.body.appendChild(link);
    link.click();
    link.remove();
}

function openLightbox(src: string | null | undefined = previewImageSrc.value, alt = "Expanded image"): void {
    if (!src) {
        return;
    }
    lightboxImageSrc.value = src;
    lightboxImageAlt.value = alt;
    lightboxOpen.value = true;
}

function closeLightbox(): void {
    lightboxOpen.value = false;
    lightboxImageSrc.value = "";
    lightboxImageAlt.value = "";
}

async function onPaste(event: ClipboardEvent): Promise<void> {
    if (!event.clipboardData?.files?.length) {
        return;
    }
    await assignImages("init_image", event.clipboardData.files);
    setMessage("Pasted image into init_image.", "success");
}

watch(baseUrl, () => { selectedRecipe.value = null; });

watch(cacheModes, (modes) => { if (!modes.includes(form.cache.mode)) form.cache.mode = "disabled"; });
watch(generationFeatures, (features) => {
    if (!features.high_noise) form.lora = form.lora.filter((item) => !item.is_high_noise);
    if (!features.ref_video_audios) videoInput.setUseAudio(false);
});

watch(generationMode, (mode) => {
    ensureOutputFormatForMode(mode);
});

watch(imageOutputFormats, () => {
    ensureOutputFormatForMode();
});

watch(videoOutputFormats, () => {
    ensureOutputFormatForMode();
});

onMounted(() => {
    window.addEventListener("paste", onPaste);
    refreshCapabilities();
});

onBeforeUnmount(() => {
    stopPolling();
    stopElapsedTimer();
    window.removeEventListener("paste", onPaste);
});
</script>

<template>
    <div class="shell">
        <header class="page-header panel">
            <div class="page-header__top">
                <div class="page-header__copy">
                    <div class="breadcrumb">
                        <span class="breadcrumb__org">stable-diffusion.cpp</span>
                        <span class="breadcrumb__slash">/</span>
                        <span>{{ modelName }}</span>
                    </div>
                    <h1 class="page-title">{{ modelName }}</h1>
                </div>
                <div class="page-header__meta">
                    <StatusChip :status="serviceOnline ? 'online' : 'offline'" :label="serviceOnline ? 'service online' : 'service unavailable'" />
                    <StatusChip :label="`queue ${queueLimit}`" />
                    <StatusChip :status="currentStatus" :label="currentStatus" />
                </div>
            </div>
            <div class="page-tabs">
                <div class="page-tabs__list">
                    <button class="page-tab" :class="{ 'page-tab--active': activeTab === 'image' }" type="button" @click="selectGenerationMode('image')" :disabled="!supportsImageMode">Image Generation</button>
                    <button class="page-tab" :class="{ 'page-tab--active': activeTab === 'video' }" type="button" @click="selectGenerationMode('video')" :disabled="!supportsVideoMode">Video Generation</button>
                    <button class="page-tab" :class="{ 'page-tab--active': activeTab === 'settings' }" type="button" @click="activeTab = 'settings'">Settings</button>
                </div>
                <div class="page-tabs__actions">
                    <button class="btn-secondary" type="button" @click="refreshCapabilities" :disabled="loadingCapabilities">{{ loadingCapabilities ? "Refreshing..." : "Refresh Server Info" }}</button>
                </div>
            </div>
            <div v-if="activeTab === 'settings'" class="settings">
                <div class="settings__grid">
                    <div class="field">
                        <label for="theme">Theme</label>
                        <select id="theme" v-model="theme">
                            <option value="system">Device</option>
                            <option value="light">Light</option>
                            <option value="dark">Dark</option>
                        </select>
                    </div>
                    <div class="field">
                        <label>Base URL</label>
                        <input v-model="baseUrl" placeholder="Leave blank to use same origin" />
                    </div>
                    <div class="field">
                        <label>Queue Limit</label>
                        <input :value="queueLimit" readonly />
                    </div>
                    <div class="field">
                        <label>Output Formats ({{ generationMode === "video" ? "video" : "image" }})</label>
                        <input :value="outputFormats.join(', ')" readonly />
                    </div>
                    <div class="field">
                        <label>Output Format</label>
                        <select v-model="form.output_format">
                            <option v-for="format in outputFormats" :key="format" :value="format">{{ format }}</option>
                        </select>
                    </div>
                    <div class="field">
                        <label>Output Compression</label>
                        <input v-model.number="form.output_compression" type="number" min="0" max="100" />
                    </div>
                    <div class="field">
                        <label>Job Poll Interval (ms)</label>
                        <input v-model.number="pollIntervalMs" type="number" min="1" step="1" />
                    </div>
                </div>
                <fieldset class="settings__group">
                    <legend>Live Preview</legend>
                    <div class="settings__grid">
                        <div class="field">
                            <label for="preview-mode">Preview Mode</label>
                            <select id="preview-mode" v-model="preferredPreviewMode">
                                <option v-for="mode in previewModes" :key="mode" :value="mode">{{ mode }}</option>
                            </select>
                        </div>
                        <div class="field">
                            <label for="preview-interval">Interval (steps)</label>
                            <input id="preview-interval" v-model.number="previewInterval" type="number" min="1" step="1" :disabled="preferredPreviewMode === 'none'" />
                        </div>
                    </div>
                    <div v-if="(selectedRecipe || modelReady) && preferredPreviewMode !== previewMode" class="hint stack-top">
                        The selected preview mode is unavailable for the current model configuration. Preview will be disabled; your preference is saved.
                    </div>
                </fieldset>
                <div v-if="capabilitiesError" class="status-message status-message--error">{{ capabilitiesError }}</div>
            </div>
        </header>

        <ModelSelector v-model:apply-defaults="applyRecipeDefaultsOnSelect" :base-url="baseUrl" :busy="isJobRunning" @status="modelStatus = $event" @changed="refreshCapabilities" @recipe="selectRecipe" @selection="modelSelection = $event" @catalog-recipe="updateCatalogRecipe" />

        <div v-if="activeTab !== 'settings'" class="layout">
            <section class="panel control-panel">
                <div class="panel-header">
                    <div>
                        <h2 class="panel-title">Input</h2>
                    </div>
                </div>

                <div class="prompt-card">
                    <div class="field--full">
                        <label>Prompt</label>
                        <textarea v-model="form.prompt" :placeholder="generationMode === 'video' ? 'Describe the motion, scene, and framing you want to generate' : 'Describe the image you want to generate'" />
                    </div>
                    <div class="field--full stack-top">
                        <label>Negative Prompt</label>
                        <textarea v-model="form.negative_prompt" placeholder="What should be excluded?" />
                    </div>
                </div>

                <div class="fields stack-top">
                    <div class="field"><label>Width</label><input v-model.number="form.width" type="number" min="64" /></div>
                    <div class="field"><label>Height</label><input v-model.number="form.height" type="number" min="64" /></div>
                </div>

                <div v-if="generationMode === 'image'" class="fields stack-top">
                    <div class="field"><label>Batch Count</label><input v-model.number="form.batch_count" type="number" min="1" /></div>
                    <div class="field">
                        <label for="image-seed">Seed</label>
                        <div class="seed-input">
                            <input id="image-seed" v-model.number="form.seed" type="number" />
                            <button class="btn-secondary" type="button" title="Generate a random seed" @click="randomizeSeed">Random</button>
                        </div>
                    </div>
                </div>
                <div v-else class="fields stack-top">
                    <div class="field"><label>Video Frames</label><input v-model.number="form.video_frames" type="number" min="1" /></div>
                    <div class="field"><label>FPS</label><input v-model.number="form.fps" type="number" min="1" /></div>
                </div>
                <div v-if="generationMode === 'video'" class="field stack-top">
                    <label for="video-seed">Seed</label>
                    <div class="seed-input">
                        <input id="video-seed" v-model.number="form.seed" type="number" />
                        <button class="btn-secondary" type="button" title="Generate a random seed" @click="randomizeSeed">Random</button>
                    </div>
                </div>

                <CollapsibleSection class="stack-top" eyebrow="Sample" :summary="sampleSummary" :open="sectionState.sample" variant="module" @toggle="toggleSection('sample')">
                    <div class="fields">
                        <div class="field">
                            <label>Scheduler</label>
                            <select v-model="form.sample_params.scheduler">
                                <option value="default">default</option>
                                <option v-for="scheduler in schedulers" :key="scheduler" :value="scheduler">{{ scheduler }}</option>
                            </select>
                        </div>
                        <div v-if="sampleHasFlowShift(form.sample_params)" class="field"><label>Flow Shift</label><input v-model="form.sample_params.flow_shift" type="number" step="0.01" placeholder="blank = default" /></div>
                        <div class="field">
                            <label>Method</label>
                            <select v-model="form.sample_params.sample_method">
                                <option value="default">default</option>
                                <option v-for="sampler in samplers" :key="sampler" :value="sampler">{{ sampler }}</option>
                            </select>
                        </div>
                        <div class="field"><label>Steps</label><input v-model.number="form.sample_params.sample_steps" type="number" /></div>
                    </div>
                    <div v-if="sampleHasExtras(form.sample_params)" class="sample-panel__extras">
                        <button class="module-card__link" type="button" @click="toggleSection('sampleAdvanced')">
                            {{ sectionState.sampleAdvanced ? "Hide extras" : "Show extras" }}
                        </button>
                    </div>
                    <div v-if="sectionState.sampleAdvanced && sampleHasExtras(form.sample_params)" class="fields">
                        <div v-if="sampleHasEta(form.sample_params)" class="field"><label>Eta</label><input v-model="form.sample_params.eta" type="number" step="0.01" placeholder="blank = default" /></div>
                        <div v-if="hasFeature('shifted_timestep')" class="field"><label>Shifted Timestep</label><input v-model.number="form.sample_params.shifted_timestep" type="number" /></div>
                    </div>
                </CollapsibleSection>

                <CollapsibleSection class="stack-top" eyebrow="Guidance" :summary="guidanceSummary" :open="sectionState.guidance" variant="module" @toggle="toggleSection('guidance')">
                    <div class="fields">
                        <div class="field"><label>CFG Scale</label><input v-model.number="form.sample_params.guidance.txt_cfg" type="number" step="0.1" /></div>
                        <div v-if="hasFeature('distilled_guidance')" class="field"><label>Distilled Guidance</label><input v-model.number="form.sample_params.guidance.distilled_guidance" type="number" step="0.1" /></div>
                    </div>
                    <div v-if="hasGuidanceExtras" class="sample-panel__extras">
                        <button class="module-card__link" type="button" @click="toggleSection('guidanceAdvanced')">
                            {{ sectionState.guidanceAdvanced ? "Hide extras" : "Show extras" }}
                        </button>
                    </div>
                    <div v-if="sectionState.guidanceAdvanced && hasGuidanceExtras" class="fields">
                        <div v-if="hasFeature('img_cfg')" class="field"><label>Image CFG</label><input v-model="form.sample_params.guidance.img_cfg" type="number" step="0.1" placeholder="blank = follow text cfg" /></div>
                        <div v-if="hasFeature('slg')" class="field"><label>SLG Layers</label><input v-model="form.sample_params.guidance.slg_layers" placeholder="7,8,9" /></div>
                        <div v-if="hasFeature('slg')" class="field"><label>SLG Layer Start</label><input v-model.number="form.sample_params.guidance.layer_start" type="number" step="0.01" /></div>
                        <div v-if="hasFeature('slg')" class="field"><label>SLG Layer End</label><input v-model.number="form.sample_params.guidance.layer_end" type="number" step="0.01" /></div>
                        <div v-if="hasFeature('slg')" class="field"><label>SLG Scale</label><input v-model.number="form.sample_params.guidance.scale" type="number" step="0.01" /></div>
                    </div>
                </CollapsibleSection>

                <CollapsibleSection v-if="hasFeature('high_noise')" class="stack-top" eyebrow="High Noise Pass" :summary="highNoiseSummary" :open="sectionState.highNoise" variant="module" @toggle="toggleSection('highNoise')">
                    <div class="field">
                        <label>MoE Boundary</label>
                        <input v-model.number="form.moe_boundary" type="number" step="0.001" />
                    </div>
                    <CollapsibleSection eyebrow="Sample" :summary="buildSampleSummary(form.high_noise_sample_params)" :open="sectionState.highNoiseSample" variant="plain" @toggle="toggleSection('highNoiseSample')">
                        <div class="fields">
                            <div class="field">
                                <label>Scheduler</label>
                                <select v-model="form.high_noise_sample_params.scheduler">
                                    <option value="default">default</option>
                                    <option v-for="scheduler in schedulers" :key="`high-noise-${scheduler}`" :value="scheduler">{{ scheduler }}</option>
                                </select>
                            </div>
                            <div v-if="sampleHasFlowShift(form.high_noise_sample_params)" class="field"><label>Flow Shift</label><input v-model="form.high_noise_sample_params.flow_shift" type="number" step="0.01" placeholder="blank = default" /></div>
                            <div class="field">
                                <label>Method</label>
                                <select v-model="form.high_noise_sample_params.sample_method">
                                    <option value="default">default</option>
                                    <option v-for="sampler in samplers" :key="`high-noise-${sampler}`" :value="sampler">{{ sampler }}</option>
                                </select>
                            </div>
                            <div class="field"><label>Steps</label><input v-model.number="form.high_noise_sample_params.sample_steps" type="number" /></div>
                            <div v-if="sampleHasEta(form.high_noise_sample_params)" class="field"><label>Eta</label><input v-model="form.high_noise_sample_params.eta" type="number" step="0.01" placeholder="blank = auto" /></div>
                            <div v-if="hasFeature('shifted_timestep')" class="field"><label>Shifted Timestep</label><input v-model.number="form.high_noise_sample_params.shifted_timestep" type="number" /></div>
                        </div>
                    </CollapsibleSection>
                    <CollapsibleSection class="stack-top" eyebrow="Guidance" :summary="buildGuidanceSummary(form.high_noise_sample_params)" :open="sectionState.highNoiseGuidance" variant="plain" @toggle="toggleSection('highNoiseGuidance')">
                        <div class="fields">
                            <div class="field"><label>CFG Scale</label><input v-model.number="form.high_noise_sample_params.guidance.txt_cfg" type="number" step="0.1" /></div>
                            <div v-if="hasFeature('distilled_guidance')" class="field"><label>Distilled Guidance</label><input v-model.number="form.high_noise_sample_params.guidance.distilled_guidance" type="number" step="0.1" /></div>
                            <div v-if="hasFeature('img_cfg')" class="field"><label>Image CFG</label><input v-model="form.high_noise_sample_params.guidance.img_cfg" type="number" step="0.1" placeholder="blank = follow text cfg" /></div>
                            <div v-if="hasFeature('slg')" class="field"><label>SLG Layers</label><input v-model="form.high_noise_sample_params.guidance.slg_layers" placeholder="7,8,9" /></div>
                            <div v-if="hasFeature('slg')" class="field"><label>SLG Layer Start</label><input v-model.number="form.high_noise_sample_params.guidance.layer_start" type="number" step="0.01" /></div>
                            <div v-if="hasFeature('slg')" class="field"><label>SLG Layer End</label><input v-model.number="form.high_noise_sample_params.guidance.layer_end" type="number" step="0.01" /></div>
                            <div v-if="hasFeature('slg')" class="field"><label>SLG Scale</label><input v-model.number="form.high_noise_sample_params.guidance.scale" type="number" step="0.01" /></div>
                        </div>
                    </CollapsibleSection>
                </CollapsibleSection>

                <CollapsibleSection v-if="hasConditioning" class="stack-top" eyebrow="Conditioning" :summary="conditioningSummary" :open="sectionState.conditioning" @toggle="toggleSection('conditioning')">
                    <div class="fields">
                        <div v-if="hasFeature('clip_skip')" class="field"><label>CLIP Skip</label><input v-model.number="form.clip_skip" type="number" /></div>
                        <div v-if="hasFeature('controlnet')" class="field"><label>Control Strength</label><input v-model.number="form.control_strength" type="number" step="0.01" /></div>
                        <div v-if="hasFeature('vace')" class="field"><label>VACE Strength</label><input v-model.number="form.vace_strength" type="number" step="0.01" /></div>
                    </div>
                </CollapsibleSection>

                <CollapsibleSection v-if="hasFeature('lora')" class="stack-top lora-section" eyebrow="LoRA" :summary="loraSummary" :open="sectionState.lora" @toggle="toggleSection('lora')">
                    <div v-if="!availableLoras.length" class="hint">No LoRA entries were returned by capabilities.</div>
                    <div v-else-if="!form.lora.length" class="hint">No LoRA overrides configured.</div>
                    <div v-else class="list-editor" :class="{ 'list-editor--single-pass': !hasFeature('high_noise') }">
                        <div class="list-row list-row--header">
                            <div>LoRA</div>
                            <div>Multiplier</div>
                            <div v-if="hasFeature('high_noise')">High Noise</div>
                            <div></div>
                        </div>
                        <div v-for="(item, index) in form.lora" :key="index" class="list-row">
                            <SearchableSelect :id="'lora-' + index" v-model="item.path" :aria-label="'LoRA ' + (index + 1)"
                                :options="availableLoras.map((lora) => ({ value: lora.path, label: lora.display_path || lora.path }))"
                                placeholder="Select a LoRA" />
                            <input v-model.number="item.multiplier" type="number" step="0.1" />
                            <label v-if="hasFeature('high_noise')" class="checkbox list-row__checkbox">
                                <input v-model="item.is_high_noise" type="checkbox" />
                                <span>{{ item.is_high_noise ? "On" : "Off" }}</span>
                            </label>
                            <button class="btn-ghost" type="button" @click="removeLora(index)">Remove</button>
                        </div>
                    </div>
                    <div><button class="btn-ghost" type="button" @click="addLora" :disabled="!availableLoras.length">Add LoRA</button></div>
                </CollapsibleSection>

                <CollapsibleSection v-if="currentImageInputs.length || hasFeature('ref_images') || hasFeature('control_frames')" class="stack-top" eyebrow="Image Inputs" :summary="imageInputsSummary" :open="sectionState.auxiliaryImages" @toggle="toggleSection('auxiliaryImages')">
                    <div v-if="gridImageInputs.length" class="upload-grid">
                        <div v-for="input in gridImageInputs" :key="input.target" class="group">
                            <ImageDropzone
                                :label="input.label"
                                :description="input.description"
                                :preview="getFormImage(input.target)"
                                @select="assignImages(input.target, $event)"
                                @clear="clearImage(input.target)"
                                @preview="openImageEntry($event)"
                            />
                            <div v-if="input.target === 'init_image' && hasFeature('img2img') && form.init_image" class="field">
                                <label for="denoising-strength">Denoising Strength</label>
                                <input id="denoising-strength" v-model.number="form.strength" type="number" min="0" max="1" step="0.01" />
                                <div class="hint">Lower values preserve more of the initial image; higher values allow larger changes.</div>
                            </div>
                        </div>
                    </div>
                    <div v-for="input in fullImageInputs" :key="input.target">
                        <ImageDropzone
                            :label="input.label"
                            :description="input.description"
                            :preview="getFormImage(input.target)"
                            @select="assignImages(input.target, $event)"
                            @clear="clearImage(input.target)"
                            @preview="openImageEntry($event)"
                        />
                        <div v-if="input.target === 'ip_adapter_image'" class="field stack-top">
                            <label for="ip-adapter-strength">IP-Adapter Strength</label>
                            <input id="ip-adapter-strength" v-model.number="form.ip_adapter_strength" type="number" min="0" step="0.01" />
                        </div>
                    </div>

                    <div v-if="generationMode === 'image' && hasFeature('ref_images')" class="group">
                        <label>Reference Images</label>
                        <ImageDropzone
                            label="Reference Images"
                            description="Multiple reference images supported."
                            :items="form.ref_images"
                            multiple
                            @select="assignImages('ref_images', $event)"
                            @clear="clearImage('ref_images')"
                        />
                        <div v-if="!form.ref_images.length" class="hint">No files selected.</div>
                        <div v-else class="file-list">
                            <div v-for="(item, index) in form.ref_images" :key="item.name + index" class="file-chip file-chip--preview">
                                <button class="file-chip__thumb-button" type="button" @click="openLightbox(item.dataUrl, item.name)">
                                    <img class="file-chip__thumb" :src="item.dataUrl" :alt="item.name" />
                                </button>
                                <span class="file-chip__name">{{ item.name }}</span>
                                <button class="icon-button" type="button" @click="removeCollectionImage('ref_images', index)">Remove</button>
                            </div>
                        </div>
                    </div>

                    <div v-if="generationMode === 'video'">
                        <div v-if="hasFeature('ref_images')" class="group">
                            <label>Reference Images</label>
                            <ImageDropzone
                                label="Reference Images"
                                description="Only MiniMax-H3 Ref2VA uses these images; other video models ignore them. Cannot be combined with an initial or end frame."
                                :items="form.ref_images"
                                multiple
                                @select="assignImages('ref_images', $event)"
                                @clear="clearImage('ref_images')"
                            />
                            <div v-if="!form.ref_images.length" class="hint">No files selected.</div>
                            <div v-else class="file-list">
                                <div v-for="(item, index) in form.ref_images" :key="item.name + index" class="file-chip file-chip--preview">
                                    <button class="file-chip__thumb-button" type="button" @click="openLightbox(item.dataUrl, item.name)">
                                        <img class="file-chip__thumb" :src="item.dataUrl" :alt="item.name" />
                                    </button>
                                    <span class="file-chip__name">{{ item.name }}</span>
                                    <button class="icon-button" type="button" @click="removeCollectionImage('ref_images', index)">Remove</button>
                                </div>
                            </div>
                        </div>
                        <div v-if="hasFeature('control_frames')" class="group stack-top">
                            <label>Control Frames</label>
                            <ImageDropzone
                                label="Control Frames"
                                description="Upload ordered conditioning frames. The server preserves the array order."
                                :items="form.control_frames"
                                multiple
                                @select="assignImages('control_frames', $event)"
                                @clear="clearImage('control_frames')"
                            />
                            <div v-if="!form.control_frames.length" class="hint">No files selected.</div>
                            <div v-else class="file-list">
                                <div v-for="(item, index) in form.control_frames" :key="item.name + index" class="file-chip file-chip--preview">
                                    <button class="file-chip__thumb-button" type="button" @click="openLightbox(item.dataUrl, item.name)">
                                        <img class="file-chip__thumb" :src="item.dataUrl" :alt="item.name" />
                                    </button>
                                    <span class="file-chip__name">{{ item.name }}</span>
                                    <button class="icon-button" type="button" @click="removeCollectionImage('control_frames', index)">Remove</button>
                                </div>
                            </div>
                        </div>
                    </div>
                </CollapsibleSection>

                <CollapsibleSection v-if="hasFeature('ref_videos') || hasFeature('ref_audios')" class="stack-top" eyebrow="Reference Media" :summary="mediaSummary" :open="sectionState.referenceMedia" @toggle="toggleSection('referenceMedia')">
                    <MediaDropzone v-if="hasFeature('ref_videos')" kind="video" :entry="form.ref_video" :allow-soundtrack="hasFeature('ref_video_audios')" @select="videoInput.select" @clear="videoInput.clear" @soundtrack="videoInput.setUseAudio" />
                    <MediaDropzone v-if="hasFeature('ref_audios')" kind="audio" :entry="form.ref_audio" @select="audioInput.select" @clear="audioInput.clear" />
                </CollapsibleSection>

                <CollapsibleSection v-if="hasFeature('vae_tiling')" class="stack-top" eyebrow="VAE Tiling" :summary="vaeTilingSummary" :open="sectionState.vaeTiling" @toggle="toggleSection('vaeTiling')">
                    <label class="checkbox"><input v-model="form.vae_tiling_params.enabled" type="checkbox" /><span>Enabled</span></label>
                    <div class="fields">
                        <div class="field"><label>Tile Width (px)</label><input v-model.number="form.vae_tiling_params.tile_size_w" type="number" /></div>
                        <div class="field"><label>Tile Height (px)</label><input v-model.number="form.vae_tiling_params.tile_size_h" type="number" /></div>
                    </div>
                    <div class="field"><label>Target Overlap</label><input v-model.number="form.vae_tiling_params.target_overlap" type="number" step="0.01" /></div>
                    <div class="fields">
                        <div class="field"><label>Relative Width</label><input v-model.number="form.vae_tiling_params.rel_size_w" type="number" step="0.01" /></div>
                        <div class="field"><label>Relative Height</label><input v-model.number="form.vae_tiling_params.rel_size_h" type="number" step="0.01" /></div>
                    </div>
                </CollapsibleSection>

                <CollapsibleSection v-if="hasFeature('cache')" class="stack-top" eyebrow="Cache" :summary="cacheSummary" :open="sectionState.cache" @toggle="toggleSection('cache')">
                    <div class="field">
                        <label>Mode</label>
                        <select v-model="form.cache.mode">
                            <option v-for="mode in cacheModes" :key="mode" :value="mode">{{ mode }}</option>
                        </select>
                    </div>
                    <div v-if="form.cache.mode !== 'disabled'" class="field field--full"><label>Cache Option</label><input v-model="form.cache.option" placeholder="threshold=0.25,start=0.15,end=0.95" /></div>
                    <div v-if="usesSCM(form.cache.mode)" class="field"><label>SCM Mask</label><input v-model="form.cache.scm_mask" /></div>
                    <label v-if="usesSCM(form.cache.mode)" class="checkbox"><input v-model="form.cache.scm_policy_dynamic" type="checkbox" /><span>Dynamic SCM Policy</span></label>
                </CollapsibleSection>
            </section>

            <section class="panel output-panel">
                <div class="panel-header">
                    <div>
                        <h2 class="panel-title">Output</h2>
                    </div>
                </div>

                <div v-if="livePreviewSrc" class="hero-frame hero-frame--media hero-frame--preview">
                    <img class="hero-frame__image" :src="livePreviewSrc" alt="Live preview" />
                    <div class="hero-frame__preview-overlay">
                        <span class="hero-frame__preview-label">Preview</span>
                        <span class="hero-frame__preview-step">Pass {{ livePreviewPass }} · {{ livePreviewStep }} / {{ livePreviewTotalSteps }} steps</span>
                    </div>
                </div>
                <div v-else-if="videoPreviewSrc" class="hero-frame hero-frame--media">
                    <video class="hero-frame__video" :src="videoPreviewSrc" controls autoplay loop muted playsinline />
                </div>
                <button v-else class="hero-frame hero-frame--button" type="button" :disabled="!previewImageSrc" @click="openLightbox(previewImageSrc, 'Generated output')">
                    <img v-if="previewImageSrc" :src="previewImageSrc" alt="Generated output" />
                    <div v-else class="hero-placeholder">
                        <h2>{{ generationMode === "video" ? "Generate Video" : "Generate Images" }}</h2>
                        <p>
                            {{ generationMode === "video"
                                ? "Generated video or animated WebP output will appear here once the current job finishes."
                                : "Generated images will appear here once the current job finishes." }}
                        </p>
                    </div>
                </button>

                <div v-if="currentJobKind === 'vid_gen' && currentJob?.result?.output_format === 'avi'" class="hint stack-top">
                    Browser playback for AVI depends on codec support. Download the file if the preview cannot play.
                </div>

                <div class="metrics output-metrics">
                    <div class="metric">
                        <div class="metric__label">Status</div>
                        <div class="metric__value">{{ currentStatus }}</div>
                    </div>
                    <div class="metric">
                        <div class="metric__label">Queue</div>
                        <div class="metric__value">{{ currentJob?.queue_position ?? 0 }}</div>
                    </div>
                    <div class="metric">
                        <div class="metric__label">Created</div>
                        <div class="metric__value mono">{{ formatUnixTime(currentJob?.created) }}</div>
                    </div>
                    <div class="metric">
                        <div class="metric__label">Elapsed</div>
                        <div class="metric__value">{{ formatElapsed(currentJob) }}</div>
                    </div>
                    <div v-if="currentJobKind === 'vid_gen'" class="metric">
                        <div class="metric__label">FPS</div>
                        <div class="metric__value">{{ videoFps || "-" }}</div>
                    </div>
                    <div v-if="currentJobKind === 'vid_gen'" class="metric">
                        <div class="metric__label">Frames</div>
                        <div class="metric__value">{{ videoFrameCount || "-" }}</div>
                    </div>
                </div>

                <div v-if="statusMessage" class="status-message" :class="statusTone === 'error' ? 'status-message--error' : 'status-message--success'">{{ statusMessage }}</div>
                <div v-else-if="currentJob?.error?.message" class="status-message status-message--error">{{ currentJob.error.message }}</div>

                <div class="output-controls">
                    <button class="btn output-controls__primary" type="button" :disabled="isJobRunning || !canGenerate || mediaProcessing || submitting" @click="generate">
                        {{ generationMode === "video" ? "Generate Video" : "Generate Image" }}
                    </button>
                    <div class="actions output-controls__secondary">
                        <button class="btn-secondary" type="button" :disabled="!downloadableSrc" @click="downloadSelected">Download</button>
                        <button class="btn-danger" type="button" :disabled="!canCancelCurrentJob" @click="cancelCurrentJob">{{ cancelling || (isJobRunning && currentJob?.cancel_requested) ? "Cancelling…" : "Cancel" }}</button>
                    </div>
                </div>

                <div v-if="currentImages.length > 1" class="thumb-row">
                    <button v-for="(image, index) in currentImages" :key="image.index" class="thumb" :class="{ 'thumb--active': index === selectedOutputIndex }" type="button" @click="selectOutput(index)">
                        <img :src="`data:image/${currentJob?.result?.output_format || 'png'};base64,${image.b64_json}`" :alt="`Output ${index + 1}`" />
                    </button>
                </div>
            </section>
        </div>
        <div v-if="lightboxOpen && lightboxImageSrc" class="lightbox" @click.self="closeLightbox">
            <button class="lightbox__close" type="button" @click="closeLightbox">Close</button>
            <img class="lightbox__image" :src="lightboxImageSrc" :alt="lightboxImageAlt" />
        </div>
    </div>
</template>
