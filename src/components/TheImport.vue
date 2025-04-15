<template>
    <div class="file-uploader-pro">
        <h2 class="uploader-title">Nhập Dữ Liệu Học Sinh Từ Excel</h2>

        <div class="school-year-group">
            <label for="schoolYearInput" class="form-label">Năm học:</label>
            <input
                type="text"
                id="schoolYearInput"
                v-model.trim="schoolYear"
                placeholder="Ví dụ: 2024 - 2025"
                class="form-control school-year-input"
                :disabled="!!selectedTypeId || isAnyLoading"
                @input="globalError = ''"
            />
            <p v-if="selectedTypeId" class="school-year-display">
                {{ schoolYear || "Chưa nhập" }}
            </p>
        </div>
        <p v-if="globalError" class="global-error-message">{{ globalError }}</p>

        <div class="content-area">
            <Transition name="grid-fade">
                <div v-if="!selectedTypeId" class="grid-container">
                    <div
                        v-for="fileType in fileTypes"
                        :key="fileType.id"
                        class="preview-card"
                        :class="{
                            'has-file': !!fileType.selectedFile,
                            'is-loading': fileType.isLoading,
                            'is-success':
                                fileType.isSuccess && !fileType.isLoading,
                            disabled: isAnyLoading && !fileType.isLoading,
                        }"
                        @click="selectType(fileType.id)"
                        tabindex="0"
                        @keydown.enter="selectType(fileType.id)"
                    >
                        <div class="card-icon">
                            <span
                                v-if="fileType.isSuccess && !fileType.isLoading"
                                >✅</span
                            >
                            <span
                                v-else-if="fileType.isLoading"
                                class="card-spinner"
                            ></span>
                            <span v-else-if="fileType.selectedFile">📄</span>
                            <span v-else>📁</span>
                        </div>
                        <h4 class="card-title">{{ fileType.name }}</h4>
                        <p class="card-status">
                            <span v-if="fileType.isLoading">Đang xử lý...</span>
                            <span v-else-if="fileType.isSuccess"
                                >Hoàn thành</span
                            >
                            <span v-else-if="fileType.selectedFile">{{
                                fileType.selectedFile.name
                            }}</span>
                            <span v-else>Chưa chọn file</span>
                        </p>
                    </div>
                </div>
            </Transition>

            <Transition name="modal-zoom" v-if="selectedTypeId">
                <div
                    v-if="selectedFileType"
                    class="modal-overlay"
                    @mousedown.self="deselectType"
                    @keydown.esc="deselectType"
                >
                    <div class="modal-content">
                        <button
                            @click.stop="deselectType"
                            class="close-btn"
                            aria-label="Đóng"
                        >
                            &times;
                        </button>
                        <h3 class="modal-title">{{ selectedFileType.name }}</h3>

                        <div class="sample-file-section">
                            <a 
                                href="#" 
                                class="sample-file-link"
                                @click.prevent="downloadSampleFile"
                            >
                                Tải file mẫu
                            </a>
                            <button 
                                class="preview-btn"
                                @click="showPreview = true"
                                aria-label="Xem mẫu file"
                                title="Xem ảnh của file"
                            >
                                <span class="preview-icon">👁️</span>
                            </button>
                        </div>

                        <div
                            class="drop-zone"
                            @dragover.prevent="
                                handleDragEvent(selectedFileType?.id, true)
                            "
                            @dragenter.prevent="
                                handleDragEvent(selectedFileType?.id, true)
                            "
                            @dragleave.prevent="
                                handleDragEvent(selectedFileType?.id, false)
                            "
                            @drop.prevent="
                                handleFileDrop($event, selectedFileType?.id)
                            "
                            @click="triggerFileInput(selectedFileType?.id)"
                            :class="{
                                'is-dragging': selectedFileType.isDragging,
                                'has-file': !!selectedFileType.selectedFile,
                                'is-disabled': selectedFileType.isLoading,
                            }"
                        >
                            <input
                                :ref="
                                    (el) =>
                                        (fileInputRefs[selectedFileType?.id] =
                                            el)
                                "
                                type="file"
                                @change="
                                    handleFileChange(
                                        $event,
                                        selectedFileType?.id
                                    )
                                "
                                accept=".xlsx, .xls"
                                hidden
                                :disabled="selectedFileType.isLoading"
                            />
                            <div class="drop-zone-content">
                                <div
                                    v-if="selectedFileType.isLoading"
                                    class="drop-zone-loading"
                                >
                                    <div class="modal-spinner"></div>
                                    <p>Đang tải lên...</p>
                                </div>
                                <template v-else>
                                    <span class="drop-zone-icon">{{
                                        selectedFileType.selectedFile
                                            ? "📄"
                                            : "📤"
                                    }}</span>
                                    <p
                                        v-if="!selectedFileType.selectedFile"
                                        class="drop-zone-text"
                                    >
                                        Kéo thả tệp vào đây hoặc
                                        <strong>nhấn để chọn</strong>
                                    </p>
                                    <p v-else class="selected-file-name">
                                        Đã chọn:
                                        <strong>{{
                                            selectedFileType.selectedFile.name
                                        }}</strong>
                                    </p>
                                    <p class="drop-zone-hint">
                                        Chấp nhận file .xls, .xlsx
                                    </p>
                                </template>
                            </div>
                        </div>

                        <button
                            class="upload-btn"
                            @click="uploadFile(selectedFileType.id)"
                            :disabled="
                                !selectedFileType.selectedFile ||
                                !schoolYear ||
                                selectedFileType.isLoading ||
                                isAnyLoading
                            "
                        >
                            <span v-if="!selectedFileType.isLoading"
                                >Tải lên</span
                            >
                            <span v-else>Đang tải...</span>
                        </button>

                        <p
                            v-if="selectedFileType.message"
                            class="status-message"
                            :class="selectedFileType.messageType"
                        >
                            {{ selectedFileType.message }}
                        </p>
                    </div>
                </div>
            </Transition>

            <!-- Preview Modal -->
            <Transition name="modal-zoom">
                <div v-if="showPreview" class="preview-modal-overlay" @click="showPreview = false">
                    <div class="preview-modal-content" @click.stop>
                        <button class="preview-close-btn" @click="showPreview = false">&times;</button>
                        <img 
                            :src="selectedFileType.templateImage" 
                            :alt="'Mẫu file ' + selectedFileType.name"
                            class="preview-image"
                        />
                    </div>
                </div>
            </Transition>
        </div>
    </div>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted, onUnmounted } from "vue";
import axios from "axios";

// --- Configuration ---
const VITE_API_BASE_URL = import.meta.env.VITE_API_BASE_URL || "/api/upload";

// --- Reactive State ---
const schoolYear = ref("");
const selectedTypeId = ref(null);
const globalError = ref("");
const fileInputRefs = reactive({});
const showPreview = ref(false);

const fileTypes = reactive([
    {
        id: 1,
        name: "File Chung Toàn Trường",
        selectedFile: null,
        isLoading: false,
        message: "",
        messageType: "",
        isDragging: false,
        isSuccess: false,
        templateImage: "/public/template_truongchung.png",
        sampleFileUrl: "https://s3-sgn10.fptcloud.com/fsel/Fsis/TRƯỜNG CHUNG.xlsx"
    },
    {
        id: 2,
        name: "File Trường Thực Nghiệm",
        selectedFile: null,
        isLoading: false,
        message: "",
        messageType: "",
        isDragging: false,
        isSuccess: false,
        templateImage: "/public/template_thucnghiem.png",
        sampleFileUrl: "https://s3-sgn10.fptcloud.com/fsel/Fsis/THỰC NGHIỆM.xlsx"
    },
    {
        id: 3,
        name: "File Chuyên Sư Phạm",
        selectedFile: null,
        isLoading: false,
        message: "",
        messageType: "",
        isDragging: false,
        isSuccess: false,
        templateImage: "/public/template_supham.png",
        sampleFileUrl: "https://s3-sgn10.fptcloud.com/fsel/Fsis/THPT Chuyen Su Pham.xlsx"
    },
    {
        id: 4,
        name: "File Lương Thế Vinh",
        selectedFile: null,
        isLoading: false,
        message: "",
        messageType: "",
        isDragging: false,
        isSuccess: false,
        templateImage: "/public/template_thevinh.png",
        sampleFileUrl: "https://s3-sgn10.fptcloud.com/fsel/Fsis/LƯƠNG THẾ VINH.xlsx"
    },
]);

// --- Computed Properties ---
const isAnyLoading = computed(() => fileTypes.some((ft) => ft.isLoading));
const selectedFileType = computed(() => {
    return selectedTypeId.value
        ? fileTypes.find((ft) => ft.id === selectedTypeId.value)
        : null;
});

// --- Helper Functions ---
const findFileType = (typeId) => fileTypes.find((ft) => ft.id === typeId);

const setFeedback = (typeId, message, type = "error") => {
    const fileType = findFileType(typeId);
    if (fileType) {
        fileType.message = message;
        fileType.messageType = type;
        fileType.isSuccess = type === "success"; // Cập nhật isSuccess dựa trên type
    }
};

const clearFeedback = (typeId) => {
    const fileType = findFileType(typeId);
    if (fileType) {
        fileType.message = "";
        fileType.messageType = "";
    }
};

const resetFileTypeState = (typeId, resetSuccess = true) => {
    const fileType = findFileType(typeId);
    if (fileType) {
        clearFeedback(typeId); // Dùng hàm clearFeedback
        fileType.isDragging = false;
        if (resetSuccess) {
            fileType.isSuccess = false;
        }
    }
};

const isValidExcelFile = (file) => {
    if (!file) return false;
    const allowedExtensions = /(\.xls|\.xlsx)$/i;
    const allowedMimeTypes = [
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    ];
    if (allowedExtensions.exec(file.name)) return true;
    return allowedMimeTypes.includes(file.type);
};

const isValidSchoolYear = (year) => {
    const pattern = /^\d{4}\s*-\s*\d{4}$/;
    if (!pattern.test(year)) return false;
    
    const [startYear, endYear] = year.split('-').map(y => parseInt(y.trim()));
    return endYear === startYear + 1;
};

const validateSchoolYear = (year) => {
    if (!year) return "Vui lòng nhập năm học";
    if (!isValidSchoolYear(year)) return "Năm học không hợp lệ. Ví dụ: 2024 - 2025";
    return "";
};

// --- Event Handlers & Actions ---
const selectType = (typeId) => {
    const yearError = validateSchoolYear(schoolYear.value);
    if (yearError) {
        globalError.value = yearError;
        window.scrollTo({ top: 0, behavior: "smooth" });
        return;
    }
    
    globalError.value = "";
    resetFileTypeState(typeId, true);
    selectedTypeId.value = typeId;
};

const deselectType = () => {
    if (selectedFileType.value?.isLoading) {
        if (confirm("Đang tải file. Bạn có chắc muốn hủy?")) {
            selectedTypeId.value = null;
        }
        return;
    }
    selectedTypeId.value = null;
};

const triggerFileInput = (typeId) => {
    if (selectedTypeId.value !== typeId || selectedFileType.value?.isLoading)
        return;
    // Đảm bảo ref tồn tại trước khi click
    const inputRef = fileInputRefs[typeId];
    if (inputRef) {
        inputRef.value = null; // Reset giá trị để có thể chọn lại cùng file
        inputRef.click();
    } else {
        console.error("Input ref not found for:", typeId);
    }
};

const processSelectedFile = (file, typeId) => {
    const fileType = findFileType(typeId);
    if (!fileType) return;

    // Reset trạng thái trước khi xử lý file mới
    resetFileTypeState(typeId, true);

    if (!file) {
        fileType.selectedFile = null;
        return;
    }

    // Kiểm tra kích thước file (max 10MB)
    const maxSize = 10 * 1024 * 1024; // 10MB
    if (file.size > maxSize) {
        setFeedback(
            typeId,
            "File quá lớn. Kích thước tối đa là 10MB."
        );
        fileType.selectedFile = null;
        return;
    }

    if (!isValidExcelFile(file)) {
        setFeedback(
            typeId,
            "Định dạng tệp không hợp lệ. Chỉ chấp nhận .xls hoặc .xlsx."
        );
        fileType.selectedFile = null;
        return;
    }

    fileType.selectedFile = file;
};

const handleFileChange = (event, typeId) => {
    if (selectedTypeId.value !== typeId) return;
    const file = event.target.files?.[0];
    processSelectedFile(file, typeId);
    // Không cần reset event.target.value vì đã làm trong triggerFileInput
};

const handleFileDrop = (event, typeId) => {
    if (selectedTypeId.value !== typeId || selectedFileType.value?.isLoading)
        return;
    const fileType = findFileType(typeId);
    if (fileType) fileType.isDragging = false;

    const file = event.dataTransfer?.files?.[0];
    processSelectedFile(file, typeId);
};

const handleDragEvent = (typeId, isEntering) => {
    if (selectedTypeId.value !== typeId || selectedFileType.value?.isLoading)
        return;
    const fileType = findFileType(typeId);
    if (fileType) {
        fileType.isDragging = isEntering;
    }
};

const uploadFile = async (typeId) => {
    if (selectedTypeId.value !== typeId) return;
    const fileType = selectedFileType.value;

    const yearError = validateSchoolYear(schoolYear.value);
    if (yearError) {
        setFeedback(typeId, yearError);
        return;
    }

    if (!fileType?.selectedFile) {
        setFeedback(typeId, "Vui lòng chọn file để tải lên.");
        return;
    }

    if (fileType.isLoading || isAnyLoading.value) {
        setFeedback(typeId, "Vui lòng chờ quá trình tải lên khác hoàn tất.");
        return;
    }

    fileType.isLoading = true;
    fileType.isSuccess = false;
    clearFeedback(typeId);

    const formData = new FormData();
    formData.append("FormFile", fileType.selectedFile);

    const uploadUrl = `${VITE_API_BASE_URL}?schoolYear=${encodeURIComponent(
        schoolYear.value
    )}&type=${encodeURIComponent(typeId*1 - 1)}`;

    try {
        const response = await axios.post(uploadUrl, formData, {
            headers: { "Content-Type": "multipart/form-data" },
            responseType: "blob",
            timeout: 300000, // 5 minutes timeout
        });

        const blob = response.data;
        const filename = `Bao_cao_hoc_tap_${schoolYear.value.trim()}.pdf`;
        
        const url = window.URL.createObjectURL(blob);
        const link = document.createElement("a");
        link.href = url;
        link.setAttribute("download", filename);
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        window.URL.revokeObjectURL(url);

        setFeedback(
            typeId,
            `Xử lý thành công! Báo cáo "${filename}" đã tải xuống.`,
            "success"
        );
        fileType.selectedFile = null;
        fileType.isSuccess = true;

    } catch (error) {
        console.error(`Upload error for ${typeId}:`, error);
        let errorMessage = `Lỗi tải lên ${fileType.name}.`;
        
        if (error.code === 'ECONNABORTED') {
            errorMessage = "Yêu cầu đã hết thời gian chờ. Vui lòng thử lại.";
        } else if (error.response) {
            if (error.response.data instanceof Blob && error.response.data.type.includes("json")) {
                try {
                    const errorJsonText = await error.response.data.text();
                    const errorJson = JSON.parse(errorJsonText);
                    errorMessage = `Lỗi ${error.response.status}: ${
                        errorJson.message || errorJson.title || errorJsonText || "Lỗi từ máy chủ"
                    }`;
                } catch (parseError) {
                    errorMessage = `Lỗi ${error.response.status}: Không thể đọc chi tiết lỗi.`;
                }
            } else if (typeof error.response.data === "string" && error.response.data.length < 500) {
                errorMessage = `Lỗi ${error.response.status}: ${error.response.data}`;
            } else {
                errorMessage = `Lỗi ${error.response.status} (${error.response.statusText || "Lỗi máy chủ không xác định"})`;
            }
        } else if (error.request) {
            errorMessage = "Không nhận được phản hồi từ máy chủ. Vui lòng kiểm tra kết nối mạng.";
        } else {
            errorMessage = `Lỗi thiết lập yêu cầu: ${error.message}`;
        }
        
        setFeedback(typeId, errorMessage, "error");
    } finally {
        const finalFileType = findFileType(typeId);
        if (finalFileType) {
            finalFileType.isLoading = false;
        }
    }
};

const downloadSampleFile = () => {
    if (!selectedFileType.value?.sampleFileUrl) return;
    
    const link = document.createElement('a');
    link.href = selectedFileType.value.sampleFileUrl;
    link.download = `mau_${selectedFileType.value.name.toLowerCase().replace(/\s+/g, '_')}.xlsx`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
};

// --- Watchers ---
watch(schoolYear, (newValue) => {
    if (newValue) {
        globalError.value = "";
    }
});
</script>

<style>
body {
    background-color: #f0f7ff;
    min-height: 100vh;
    width: 100%;
    margin: 0;
    font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
}

#app {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 90%;
    height: 90vh;
    background-color: #fff;
    padding: 1rem;
    box-sizing: border-box;
    border-radius: 16px;
}

/* === CSS Variables === */
:root {
    --primary-gradient: linear-gradient(135deg, #4f46e5, #7c3aed);
    --primary-color: #4f46e5;
    --primary-color-dark: #4338ca;
    --success-gradient: linear-gradient(135deg, #10b981, #059669);
    --success-color: #10b981;
    --error-gradient: linear-gradient(135deg, #ef4444, #dc2626);
    --error-color: #ef4444;
    --warning-gradient: linear-gradient(135deg, #f59e0b, #d97706);
    --warning-color: #f59e0b;
    --light-bg: #f8fafc;
    --white-color: #ffffff;
    --dark-text: #1e293b;
    --grey-text: #64748b;
    --light-grey-text: #94a3b8;
    --border-color: #e2e8f0;
    --card-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    --card-hover-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
    --modal-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
    --border-radius: 1rem;
    --font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

/* === Base Styles === */
.file-uploader-pro {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 1.5rem;
    display: flex;
    flex-direction: column;
}

.uploader-title {
    text-align: center;
    color: var(--dark-text);
    margin-bottom: 1.5rem;
    font-weight: 700;
    font-size: 2rem;
    background: var(--primary-gradient);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
}

/* === School Year Input === */
.school-year-group {
    max-width: 500px;
    margin: 0 auto 1.5rem auto;
    text-align: left;
}

.form-label {
    display: block;
    margin-bottom: 0.75rem;
    font-weight: 600;
    color: var(--grey-text);
    font-size: 1rem;
}

.form-control {
    width: 100%;
    padding: 0.875rem 1.25rem;
    border: 2px solid var(--border-color);
    border-radius: var(--border-radius);
    font-size: 1rem;
    color: var(--dark-text);
    background-color: var(--white-color);
    transition: all 0.3s ease;
}

.form-control:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.1);
    outline: none;
}

.form-control:disabled {
    background-color: var(--light-bg);
    opacity: 0.7;
    cursor: not-allowed;
}

.school-year-display {
    padding: 0.875rem 1.25rem;
    background: var(--light-bg);
    border: 2px solid var(--border-color);
    border-radius: var(--border-radius);
    color: var(--grey-text);
    font-size: 1rem;
    margin-top: 0.75rem;
}

.global-error-message {
    text-align: center;
    color: var(--error-color);
    margin: 0 auto 1.5rem auto;
    font-weight: 500;
    font-size: 0.95rem;
    padding: 0.75rem 1.25rem;
    background: rgba(239, 68, 68, 0.1);
    border-radius: var(--border-radius);
    border: 1px solid rgba(239, 68, 68, 0.2);
    max-width: 500px;
    width: 100%;
    box-sizing: border-box;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
}

.global-error-message::before {
    content: "⚠️";
    font-size: 1.1rem;
}

/* === Grid View === */
.grid-container {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1.5rem;
    margin: 0 auto;
    max-width: 100%;
    padding: 1rem 2rem;
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none; /* Firefox */
    -ms-overflow-style: none; /* IE and Edge */
    position: relative;
    flex: 1;
}

.grid-container::-webkit-scrollbar {
    display: none; /* Chrome, Safari, Opera */
}

.preview-card {
    background: var(--white-color);
    border: 2px solid var(--border-color);
    border-radius: var(--border-radius);
    padding: 1.5rem;
    text-align: center;
    cursor: pointer;
    transition: all 0.3s ease;
    position: relative;
    overflow: visible;
    box-shadow: var(--card-shadow);
    min-width: 160px;
    aspect-ratio: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    margin-bottom: 0.5rem;
}

.preview-card:hover:not(.disabled) {
    transform: translateY(-5px);
    box-shadow: var(--card-hover-shadow);
    border-color: var(--primary-color);
    z-index: 1;
}

.preview-card.disabled {
    opacity: 0.6;
    cursor: not-allowed;
    background-color: var(--light-bg);
}

.card-icon {
    font-size: 2rem;
    margin-bottom: 0.75rem;
    display: block;
    color: var(--primary-color);
    height: 2.5rem;
    transition: transform 0.3s ease;
}

.preview-card:hover:not(.disabled) .card-icon {
    transform: scale(1.1);
}

.card-title {
    font-size: 1.1rem;
    font-weight: 600;
    margin: 0 0 0.5rem 0;
    color: var(--dark-text);
}

.card-status {
    font-size: 0.85rem;
    color: var(--grey-text);
    min-height: 1.2em;
    word-break: break-word;
    padding: 0 0.5rem;
    line-height: 1.4;
}

.preview-card.has-file .card-icon {
    color: var(--primary-color);
}

.preview-card.is-success .card-icon {
    color: var(--success-color);
}

.preview-card.is-success .card-status {
    color: var(--success-color);
    font-weight: 500;
}

.preview-card.is-loading .card-status {
    color: var(--primary-color-dark);
    font-weight: 500;
}

.card-spinner {
    display: inline-block;
    border: 3px solid rgba(79, 70, 229, 0.2);
    border-left-color: var(--primary-color);
    border-radius: 50%;
    width: 2rem;
    height: 2rem;
    animation: spin 1s linear infinite;
    vertical-align: middle;
}

/* === Modal View === */
.modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(30, 41, 59, 0.7);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1050;
    padding: 1.5rem;
    box-sizing: border-box;
    backdrop-filter: blur(8px);
    transition: opacity 0.3s ease;
}

.modal-content {
    background-color: var(--white-color);
    padding: 2.5rem;
    border-radius: var(--border-radius);
    box-shadow: var(--modal-shadow);
    max-width: 600px;
    width: 100%;
    position: relative;
    max-height: 90vh;
    overflow-y: auto;
    text-align: center;
    border: none;
    transform-origin: center;
    transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.close-btn {
    position: absolute;
    top: 1rem;
    right: 1.25rem;
    background: transparent;
    border: none;
    font-size: 1.75rem;
    font-weight: 700;
    line-height: 1;
    color: var(--grey-text);
    opacity: 0.7;
    cursor: pointer;
    padding: 0.5rem;
    margin: -0.5rem;
    transition: all 0.2s ease;
    border-radius: 50%;
}

.close-btn:hover {
    color: var(--error-color);
    opacity: 1;
    background-color: rgba(239, 68, 68, 0.1);
}

.close-btn:active {
    transform: scale(0.95);
}

.modal-title {
    font-size: 1.75rem;
    font-weight: 700;
    margin: 0 0 2rem 0;
    color: var(--dark-text);
    background: var(--primary-gradient);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
}

/* Drop Zone Styling */
.drop-zone {
    border: 2px dashed var(--border-color);
    border-radius: var(--border-radius);
    padding: 2.5rem 2rem;
    margin-bottom: 2rem;
    background-color: var(--white-color);
    cursor: pointer;
    transition: all 0.3s ease;
    position: relative;
}

.drop-zone:hover:not(.is-disabled) {
    border-color: var(--primary-color);
    background-color: rgba(79, 70, 229, 0.02);
}

.drop-zone.is-dragging {
    border-color: var(--primary-color);
    background-color: rgba(79, 70, 229, 0.05);
    transform: scale(1.01);
}

.drop-zone.has-file:not(.is-dragging) {
    border-style: solid;
    border-color: var(--success-color);
    background-color: rgba(16, 185, 129, 0.05);
}

.drop-zone.is-disabled {
    cursor: not-allowed;
    opacity: 0.7;
}

.drop-zone-content {
    color: var(--grey-text);
}

.drop-zone-icon {
    font-size: 2.5rem;
    margin-bottom: 1rem;
    display: block;
    color: var(--primary-color);
    transition: transform 0.3s ease;
}

.drop-zone:hover:not(.is-disabled) .drop-zone-icon {
    transform: scale(1.1);
}

.drop-zone-text strong {
    color: var(--primary-color);
    font-weight: 600;
}

.selected-file-name strong {
    display: inline-block;
    max-width: 95%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    vertical-align: bottom;
    color: var(--dark-text);
}

.drop-zone-hint {
    font-size: 0.875rem;
    color: var(--light-grey-text);
    margin-top: 1rem !important;
}

.drop-zone-loading {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(255, 255, 255, 0.9);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    border-radius: var(--border-radius);
    z-index: 2;
    backdrop-filter: blur(4px);
}

.drop-zone-loading p {
    margin-top: 1rem;
    font-weight: 500;
    color: var(--primary-color-dark);
}

.modal-spinner {
    border: 4px solid rgba(79, 70, 229, 0.2);
    border-left-color: var(--primary-color);
    border-radius: 50%;
    width: 2.5rem;
    height: 2.5rem;
    animation: spin 1s linear infinite;
    box-shadow: 0 0 20px rgba(79, 70, 229, 0.2);
}

/* Upload Button */
.upload-btn {
    background: var(--primary-gradient);
    color: var(--white-color);
    border: none;
    padding: 0.875rem 1.5rem;
    border-radius: var(--border-radius);
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    min-width: 160px;
    position: relative;
    overflow: hidden;
    box-shadow: 0 4px 6px rgba(79, 70, 229, 0.1);
}

.upload-btn:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 6px 12px rgba(79, 70, 229, 0.15);
}

.upload-btn:active:not(:disabled) {
    transform: translateY(0);
    box-shadow: 0 2px 4px rgba(79, 70, 229, 0.1);
}

.upload-btn:disabled {
    background: var(--light-grey-text);
    cursor: not-allowed;
    opacity: 0.65;
}

/* Status Message */
.status-message {
    margin-top: 1.5rem;
    padding: 0.875rem 1.25rem;
    border-radius: var(--border-radius);
    font-size: 0.95rem;
    text-align: left;
    word-wrap: break-word;
    border: 1px solid transparent;
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.status-message::before {
    font-size: 1.1rem;
}

.status-message.success {
    color: #065f46;
    background-color: rgba(16, 185, 129, 0.1);
    border-color: rgba(16, 185, 129, 0.2);
}

.status-message.success::before {
    content: "✅";
}

.status-message.error {
    color: #991b1b;
    background-color: rgba(239, 68, 68, 0.1);
    border-color: rgba(239, 68, 68, 0.2);
}

.status-message.error::before {
    content: "⚠️";
}

.status-message.warning {
    color: #92400e;
    background-color: rgba(245, 158, 11, 0.1);
    border-color: rgba(245, 158, 11, 0.2);
}

.status-message.warning::before {
    content: "⚠️";
}

/* === Animations === */
@keyframes spin {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

.grid-fade-enter-active,
.grid-fade-leave-active {
    transition: opacity 0.3s ease;
}

.grid-fade-enter-from,
.grid-fade-leave-to {
    opacity: 0;
}

.modal-zoom-enter-active,
.modal-zoom-leave-active {
    transition: opacity 0.3s ease;
}

.modal-zoom-enter-from,
.modal-zoom-leave-to {
    opacity: 0;
}

.modal-zoom-enter-active .modal-content,
.modal-zoom-leave-active .modal-content {
    transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1), opacity 0.3s ease;
}

.modal-zoom-enter-from .modal-content,
.modal-zoom-leave-to .modal-content {
    transform: scale(0.95);
    opacity: 0;
}

/* === Responsive === */
@media (max-width: 1600px) {
    .file-uploader-pro {
        padding: 1rem;
    }

    .grid-container {
        gap: 1.25rem;
        padding: 1rem 1.5rem;
    }
}

@media (max-width: 768px) {
    .file-uploader-pro {
        padding: 0.75rem;
    }

    .uploader-title {
        font-size: 1.5rem;
        margin-bottom: 1rem;
    }

    .grid-container {
        gap: 1rem;
        padding: 1rem;
    }

    .preview-card {
        min-width: 140px;
        padding: 1rem;
    }

    .card-icon {
        font-size: 1.5rem;
        margin-bottom: 0.5rem;
        height: 1.75rem;
    }

    .card-title {
        font-size: 0.9rem;
    }

    .card-status {
        font-size: 0.75rem;
    }

    .modal-content {
        padding: 1.5rem;
        max-width: 95%;
    }

    .modal-title {
        font-size: 1.5rem;
    }

    .close-btn {
        font-size: 1.5rem;
        top: 0.75rem;
        right: 0.75rem;
    }

    .school-year-group {
        max-width: none;
    }

    .drop-zone {
        padding: 1.5rem;
    }

    .upload-btn {
        padding: 0.75rem 1.25rem;
        font-size: 0.95rem;
    }

    .global-error-message {
        font-size: 0.9rem;
        padding: 0.75rem 1rem;
        margin: 0 auto 1rem auto;
    }

    .status-message {
        font-size: 0.9rem;
        padding: 0.75rem 1rem;
        margin-top: 1rem;
    }
}

/* Sample File Section */
.sample-file-section {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    margin-bottom: 1.5rem;
    padding: 0.75rem;
    background: rgba(79, 70, 229, 0.05);
    border-radius: var(--border-radius);
}

.sample-file-link {
    color: var(--primary-color);
    text-decoration: none;
    font-size: 0.95rem;
    font-weight: 500;
    transition: all 0.2s ease;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
}

.sample-file-link:hover {
    color: var(--primary-color-dark);
    background: rgba(79, 70, 229, 0.1);
}

.preview-btn {
    background: none;
    border: none;
    padding: 0.5rem;
    cursor: pointer;
    border-radius: 50%;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
}

.preview-btn:hover {
    background: rgba(79, 70, 229, 0.1);
    transform: scale(1.1);
}

.preview-icon {
    font-size: 1.25rem;
}

/* Preview Modal */
.preview-modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1060;
    padding: 1rem;
}

.preview-modal-content {
    position: relative;
    min-width: 50%;
    min-height: 50%;
    max-width: 90%;
    max-height: 90vh;
    background: var(--white-color);
    border-radius: var(--border-radius);
    overflow: hidden;
    box-shadow: var(--modal-shadow);
    display: flex;
    align-items: center;
    justify-content: center;
}

.preview-close-btn {
    position: absolute;
    top: 0.75rem;
    right: 0.75rem;
    background: rgba(0, 0, 0, 0.5);
    color: white;
    border: none;
    width: 2rem;
    height: 2rem;
    border-radius: 50%;
    font-size: 1.25rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
}

.preview-close-btn:hover {
    background: rgba(0, 0, 0, 0.7);
    transform: scale(1.1);
}

.preview-image {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
    display: block;
}

/* Responsive */
@media (max-width: 768px) {
    .sample-file-section {
        padding: 0.5rem;
        margin-bottom: 1rem;
    }

    .preview-btn {
        padding: 0.4rem;
    }

    .preview-icon {
        font-size: 1.1rem;
    }

    .preview-modal-content {
        min-width: 90%;
        min-height: 50%;
    }
}
</style>
