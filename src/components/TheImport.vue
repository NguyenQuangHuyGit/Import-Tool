<template>
    <div class="excel-uploader-container">
        <h2>Tải lên Tệp Excel</h2>

        <div class="form-group">
            <label for="schoolYearInput">Nhập năm học:</label>
            <input
                type="text"
                id="schoolYearInput"
                v-model="schoolYear"
                placeholder="Ví dụ: 2024 - 2025"
                class="form-control"
                :disabled="isLoading"
            />
        </div>
        <div
            class="file-drop-area"
            @click="triggerFileInput"
            @dragover.prevent="isDragging = true"
            @dragenter.prevent="isDragging = true"
            @dragleave.prevent="isDragging = false"
            @drop.prevent="handleFileDrop"
            :class="{ 'is-dragging': isDragging, 'has-file': !!selectedFile }"
        >
            <input
                type="file"
                ref="fileInput"
                @change="handleFileChange"
                accept=".xlsx, .xls"
                hidden
                :disabled="isLoading"
            />
            <span v-if="!selectedFile"
                >Kéo và thả tệp vào đây hoặc Nhấn để chọn</span
            >
            <span v-else>
                <span class="file-icon">📄</span> Đã chọn:
                <strong>{{ selectedFile.name }}</strong>
            </span>
        </div>

        <button
            class="upload-button"
            @click="uploadFile"
            :disabled="!selectedFile || !schoolYear || isLoading"
        >
            <span v-if="!isLoading">Tải lên</span>
            <span v-else>Đang tải...</span>
        </button>

        <div v-if="isLoading" class="loading-overlay">
            <div class="spinner"></div>
            <p>Đang xử lý, vui lòng chờ...</p>
        </div>

        <div v-if="uploadMessage" :class="['message', messageType]">
            {{ uploadMessage }}
        </div>
    </div>
</template>

<script setup>
import { ref } from "vue";
import axios from "axios";

// --- State Variables ---
const fileInput = ref(null);
const selectedFile = ref(null);
const isLoading = ref(false);
const uploadMessage = ref("");
const messageType = ref("");
const isDragging = ref(false);
const schoolYear = ref("");

let api = "https://5sis-gateway-testing.fsel.edu.vn/learning-gateway/v1/report/export-semester-result-report";

const triggerFileInput = () => {
    if (!isLoading.value) {
        fileInput.value?.click();
    }
};

const handleFileChange = (event) => {
    isDragging.value = false; 
    const file = event.target.files?.[0];
    if (file) {
        processSelectedFile(file);
    }
    event.target.value = null;
};

const handleFileDrop = (event) => {
    isDragging.value = false;
    const file = event.dataTransfer?.files?.[0];
    if (file && !isLoading.value) {
        if (isValidExcelFile(file)) {
            processSelectedFile(file);
        } else {
            setFeedback("Chỉ chấp nhận tệp .xls hoặc .xlsx", "error");
            selectedFile.value = null;
        }
    }
};

const isValidExcelFile = (file) => {
    const allowedTypes = [
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    ];
    const allowedExtensions = [".xls", ".xlsx"];
    const fileExtension = "." + file.name.split(".").pop()?.toLowerCase();
    return (
        allowedTypes.includes(file.type) ||
        allowedExtensions.includes(fileExtension)
    );
};

const processSelectedFile = (file) => {
    selectedFile.value = file;
    clearFeedback();
};

const uploadFile = async () => {
    if (!selectedFile.value || !schoolYear.value || isLoading.value) {
        if (!schoolYear.value) {
            setFeedback("Vui lòng nhập năm học.", "error");
        } else if (!selectedFile.value) {
            setFeedback("Vui lòng chọn tệp Excel.", "error");
        }
        return;
    }

    isLoading.value = true;
    clearFeedback();

    const formData = new FormData();
    formData.append("FormFile", selectedFile.value);

    if (schoolYear.value) {
        api += "?schoolYear=" + schoolYear.value
    }

    try {
        const response = await axios.post(api, formData, {
            headers: {
                "Content-Type": "multipart/form-data",
            },
            responseType: 'blob'
        });
        const blob = response.data;
        let filename = `Bao_cao_ket_qua_${schoolYear.value}.zip`;
        const contentDisposition = response.headers['content-disposition'];
        if (contentDisposition) {
            const filenameMatch = contentDisposition.match(/filename\*?=['"]?([^'";]+)['"]?/);
            if (filenameMatch && filenameMatch[1]) {
                filename = decodeURIComponent(filenameMatch[1]);
            }
        }

        const url = window.URL.createObjectURL(blob);

        const link = document.createElement('a');
        link.href = url;
        link.setAttribute('download', filename);
        document.body.appendChild(link); 

        link.click();

        link.parentNode.removeChild(link);
        window.URL.revokeObjectURL(url);

        setFeedback(
            `Tải tệp báo cáo điểm học sinh cho năm học ${schoolYear.value} thành công!`,
            "success"
        );
        selectedFile.value = null;
    } catch (error) {
        let errorMessage = "Đã xảy ra lỗi khi tải tệp lên.";
        if (error.response) {
            errorMessage = `Lỗi ${error.response.status}: ${
                error.response.data?.message || "Lỗi từ máy chủ"
            }`;
        } else if (error.request) {
            errorMessage = "Không nhận được phản hồi từ máy chủ.";
        }
        setFeedback(errorMessage, "error");
    } finally {
        isLoading.value = false;
    }
};

const setFeedback = (message, type) => {
    uploadMessage.value = message;
    messageType.value = type;
};

const clearFeedback = () => {
    uploadMessage.value = "";
    messageType.value = "";
};
</script>

<style scoped>
.excel-uploader-container {
    max-width: 500px;
    margin: 40px auto;
    padding: 30px 35px;
    background-color: #f9f9f9;
    border-radius: 12px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
    text-align: center;
    font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
    position: relative;
    overflow: hidden;
}

h2 {
    color: #333;
    margin-bottom: 30px;
    font-weight: 600;
}

.form-group {
    margin-bottom: 25px;
    text-align: left;
}

.form-group label {
    display: block;
    margin-bottom: 8px;
    color: #555;
    font-size: 0.95em;
    font-weight: 500;
}

.form-control {
    width: 100%;
    padding: 10px 14px;
    border: 1px solid #ccc;
    border-radius: 6px;
    font-size: 1em;
    color: #333;
    box-sizing: border-box;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.form-control:focus {
    border-color: #007bff;
    box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.15);
    outline: none;
}

.form-control:disabled {
    background-color: #e9ecef;
    opacity: 0.7;
    cursor: not-allowed;
}

.file-drop-area {
    border: 2px dashed #ccc;
    border-radius: 8px;
    padding: 35px 20px;
    margin-bottom: 25px;
    cursor: pointer;
    background-color: #fff;
    transition: border-color 0.3s ease, background-color 0.3s ease;
    color: #666;
    position: relative;
}

.file-drop-area.is-dragging {
    border-color: #007bff;
    background-color: #f0f8ff;
}

.file-drop-area.has-file {
    border-style: solid;
    border-color: #a5d6a7;
    background-color: #e8f5e9;
}

.file-drop-area span {
    display: block;
    font-size: 1em;
}

.file-icon {
    font-size: 1.2em;
    margin-right: 8px;
    vertical-align: middle;
}

.file-drop-area strong {
    color: #333;
    font-weight: 500;
}

.upload-button {
    background-color: #007bff;
    color: white;
    border: none;
    padding: 12px 30px;
    border-radius: 6px;
    font-size: 1.05em;
    cursor: pointer;
    transition: background-color 0.3s ease, opacity 0.3s ease;
    min-width: 130px;
    font-weight: 500;
}

.upload-button:hover:not(:disabled) {
    background-color: #0056b3;
}

.upload-button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
    opacity: 0.7;
}

.loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(255, 255, 255, 0.85);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 10;
    backdrop-filter: blur(3px);
}

.spinner {
    border: 5px solid #f3f3f3;
    border-top: 5px solid #007bff;
    border-radius: 50%;
    width: 50px;
    height: 50px;
    animation: spin 1s linear infinite;
    margin-bottom: 15px;
}

.loading-overlay p {
    color: #333;
    font-size: 1.1em;
}

@keyframes spin {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

.message {
    margin-top: 25px;
    padding: 12px 18px;
    border-radius: 6px;
    font-size: 0.95em;
    text-align: left;
}

.message.success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.message.error {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
}
</style>
