<script setup>
import { ref, computed, watch, onMounted } from 'vue';
import { useToast } from 'primevue/usetoast';

// Khởi tạo toast service
const toast = useToast();

// Khôi phục dữ liệu từ localStorage nếu có
const loadFromLocalStorage = () => {
    const savedTasks = localStorage.getItem('todo-tasks');
    return savedTasks ? JSON.parse(savedTasks) : [
        { id: 1, text: 'Học Vue.js', completed: false, priority: 'medium', createdAt: new Date().toISOString() },
        { id: 2, text: 'Học PrimeVue', completed: false, priority: 'high', createdAt: new Date().toISOString() },
        { id: 3, text: 'Xây dựng To-Do List', completed: true, priority: 'low', createdAt: new Date().toISOString() }
    ];
};

// Dữ liệu
const tasks = ref(loadFromLocalStorage());
const newTask = ref('');
const editingTask = ref(null);
const editText = ref('');
const filter = ref('all');
const priorityFilter = ref('all');
const sortBy = ref('createdAt');
const searchQuery = ref('');
const darkMode = ref(false);
// Thêm state để quản lý việc chọn nhiều task
const selectedTasks = ref([]);



const sortOptions = [
    { label: 'Mới nhất', value: 'createdAt' },
    { label: 'Cũ nhất', value: 'oldest' },
    { label: 'A-Z', value: 'alphabetical' },
];

const priorityClass = (priority) => {
    switch (priority) {
        case 'high': return 'priority-high';
        case 'medium': return 'priority-medium';
        case 'low': return 'priority-low';
        default: return '';
    }
};

// ID tăng dần cho task mới
const nextId = computed(() => {
    return tasks.value.length ? Math.max(...tasks.value.map(t => t.id)) + 1 : 1;
});

// Lưu vào localStorage khi tasks thay đổi
watch(tasks, (newTasks) => {
    localStorage.setItem('todo-tasks', JSON.stringify(newTasks));
}, { deep: true });



// Các computed properties
const filteredAndSortedTasks = computed(() => {
    // Bước 1: Lọc theo trạng thái hoàn thành
    let result = tasks.value;
    if (filter.value === 'active') {
        result = result.filter(task => !task.completed);
    } else if (filter.value === 'completed') {
        result = result.filter(task => task.completed);
    }

    // Bước 2: Lọc theo mức độ ưu tiên
    if (priorityFilter.value !== 'all') {
        result = result.filter(task => task.priority === priorityFilter.value);
    }

    // Bước 3: Lọc theo từ khóa tìm kiếm
    if (searchQuery.value.trim()) {
        const query = searchQuery.value.toLowerCase();
        result = result.filter(task => task.text.toLowerCase().includes(query));
    }

    // Bước 4: Sắp xếp
    return result.slice().sort((a, b) => {
        switch (sortBy.value) {
            case 'createdAt':
                return new Date(b.createdAt) - new Date(a.createdAt);
            case 'oldest':
                return new Date(a.createdAt) - new Date(b.createdAt);
            case 'alphabetical':
                return a.text.localeCompare(b.text);
            case 'priority':
                {
                    const priorityOrder = { high: 1, medium: 2, low: 3 };
                    return priorityOrder[a.priority] - priorityOrder[b.priority];
                }
            default:
                return 0;
        }
    });
});


const completedTasks = computed(() => {
    return tasks.value.filter(task => task.completed).length;
});



// Các methods
const addTask = () => {
    if (newTask.value.trim()) {
        const taskText = newTask.value.trim();
        const newTaskObj = {
            id: nextId.value,
            text: taskText,
            completed: false,
            priority: 'low', // Mức ưu tiên mặc định
            createdAt: new Date().toISOString()
        };

        tasks.value.push(newTaskObj);
        newTask.value = '';

        toast.add({
            severity: 'success',
            summary: 'Thêm thành công',
            detail: `Đã thêm công việc "${taskText}"`,
            life: 3000
        });
    }
};

const startEditing = (task) => {
    editingTask.value = task.id;
    editText.value = task.text;
};

const saveEdit = (task) => {
    const trimmedText = editText.value.trim();
    if (trimmedText) {
        const index = tasks.value.findIndex(t => t.id === task.id);
        if (index !== -1) {
            tasks.value[index].text = trimmedText;
            toast.add({
                severity: 'info',
                summary: 'Cập nhật thành công',
                detail: 'Đã cập nhật nội dung công việc',
                life: 3000
            });
        }
    }
    editingTask.value = null;
};

const cancelEdit = () => {
    editingTask.value = null;
};

const deleteTask = (id) => {
    const index = tasks.value.findIndex(task => task.id === id);
    if (index !== -1) {
        const taskText = tasks.value[index].text;
        tasks.value.splice(index, 1);

        toast.add({
            severity: 'warn',
            summary: 'Đã xóa',
            detail: `Đã xóa công việc "${taskText}"`,
            life: 3000
        });
    }
};

const toggleTaskStatus = (task) => {
    task.completed = !task.completed;

    if (task.completed) {
        toast.add({
            severity: 'success',
            summary: 'Hoàn thành',
            detail: `Đã hoàn thành công việc "${task.text}"`,
            life: 2000
        });
    }
};

const updatePriority = (task, priority) => {
    task.priority = priority;
};

const clearCompleted = () => {
    const completedCount = tasks.value.filter(task => task.completed).length;
    tasks.value = tasks.value.filter(task => !task.completed);

    toast.add({
        severity: 'info',
        summary: 'Đã xóa công việc hoàn thành',
        detail: `Đã xóa ${completedCount} công việc đã hoàn thành`,
        life: 3000
    });
};

// Thêm phương thức đánh dấu hoàn thành cho tất cả
const markAllAsCompleted = () => {
    const uncompletedCount = tasks.value.filter(task => !task.completed).length;

    tasks.value.forEach(task => {
        if (!task.completed) {
            task.completed = true;
        }
    });

    if (uncompletedCount > 0) {
        toast.add({
            severity: 'success',
            summary: 'Hoàn thành tất cả',
            detail: `Đã đánh dấu ${uncompletedCount} công việc là hoàn thành`,
            life: 3000
        });
    }
};

// Thêm phương thức xử lý xóa nhiều task đã chọn
const deleteSelectedTasks = () => {
    if (selectedTasks.value.length === 0) return;

    const count = selectedTasks.value.length;
    tasks.value = tasks.value.filter(task => !selectedTasks.value.includes(task.id));
    selectedTasks.value = [];

    toast.add({
        severity: 'warn',
        summary: 'Đã xóa',
        detail: `Đã xóa ${count} công việc đã chọn`,
        life: 3000
    });
};

// Kiểm tra xem có task nào được chọn không
const hasSelectedTasks = computed(() => {
    return selectedTasks.value.length > 0;
});

// Chọn/bỏ chọn tất cả tasks hiện tại
const selectAllVisible = ref(false);
const toggleSelectAllVisible = () => {
    if (selectAllVisible.value) {
        // Bỏ chọn tất cả
        selectedTasks.value = [];
    } else {
        // Chọn tất cả các task hiển thị
        selectedTasks.value = filteredAndSortedTasks.value.map(task => task.id);
    }
    selectAllVisible.value = !selectAllVisible.value;
};
</script>

<template>
    <Toast />
    <div class="todo-app">
        <Card class="todo-card">
            <template #header>
                <div class="todo-header">
                    <h1 class="todo-title">To-Do List</h1>
                </div>
            </template>

            <template #content>
                <!-- Thêm công việc mới -->
                <div class="add-task-container">
                    <span class="p-input-icon-left p-input-icon-right flex-grow-1">
                        <InputText v-model="newTask" placeholder="Thêm công việc mới..." class="task-input w-full"
                            @keyup.enter="addTask" />
                    </span>
                    <Button icon="pi pi-plus" class="add-button p-button-rounded" @click="addTask"
                        :disabled="!newTask.trim()" v-tooltip="'Thêm công việc'" />
                </div>

                <!-- Tìm kiếm và lọc -->
                <div class="search-filter-container">
                    <span class="p-input-icon-left search-box">
                        <InputText v-model="searchQuery" placeholder="Tìm kiếm..." class="w-full" />
                    </span>

                    <div class="filter-sort-section">
                        <Dropdown v-model="sortBy" :options="sortOptions" optionLabel="label" optionValue="value"
                            placeholder="Sắp xếp" class="sort-dropdown" />
                    </div>
                </div>



                <!-- Thanh công cụ xử lý hàng loạt -->
                <div class="batch-actions" v-if="tasks.length > 0">

                    <div class="batch-actions-right">
                        <Button icon="pi pi-check-circle" label="Đánh dấu hoàn thành"
                            class="p-button-success p-button-sm" @click="markAllAsCompleted"
                            v-tooltip="'Đánh dấu tất cả là hoàn thành'" />
                        <Button icon="pi pi-trash" label="Xóa đã chọn" class="p-button-danger p-button-sm"
                            @click="deleteSelectedTasks" :disabled="!hasSelectedTasks"
                            v-tooltip="'Xóa các mục đã chọn'" />
                    </div>
                </div>

                <!-- Danh sách công việc -->
                <div class="tasks-container">
                    <TransitionGroup name="list" tag="ul" class="task-list">
                        <li v-for="task in filteredAndSortedTasks" :key="task.id" class="task-item" :class="[
                            { 'completed-task': task.completed },
                            priorityClass(task.priority)
                        ]">
                            <div class="task-content">
                                <!-- Checkbox chọn task -->
                                <Checkbox v-model="selectedTasks" :value="task.id" class="select-task-checkbox" />



                                <div v-if="editingTask === task.id" class="edit-container">
                                    <InputText v-model="editText" class="edit-input" @keyup.enter="saveEdit(task)"
                                        autofocus />
                                    <div class="edit-actions">
                                        <Button icon="pi pi-check" class="p-button-rounded p-button-success p-button-sm"
                                            @click="saveEdit(task)" v-tooltip="'Lưu'" />
                                        <Button icon="pi pi-times"
                                            class="p-button-rounded p-button-secondary p-button-sm" @click="cancelEdit"
                                            v-tooltip="'Hủy'" />
                                    </div>
                                </div>

                                <div v-else class="task-text-container">
                                    <span :class="['task-text', { 'completed': task.completed }]"
                                        @click="toggleTaskStatus(task)">
                                        {{ task.text }}
                                    </span>
                                    <small class="task-date">{{ new Date(task.createdAt).toLocaleDateString() }}</small>
                                </div>
                            </div>

                            <div class="task-actions">
                                <Menu :model="[
                                    {
                                        label: 'Ưu tiên',
                                        items: [
                                            { label: 'Cao', icon: 'pi pi-flag-fill', command: () => updatePriority(task, 'high') },
                                            { label: 'Trung bình', icon: 'pi pi-flag', command: () => updatePriority(task, 'medium') },
                                            { label: 'Thấp', icon: 'pi pi-bookmark', command: () => updatePriority(task, 'low') }
                                        ]
                                    }
                                ]" popup ref="priorityMenu" />

                                <!-- Nút đánh dấu hoàn thành -->
                                <Button icon="pi pi-check-circle"
                                    class="p-button-rounded p-button-text p-button-success complete-button"
                                    @click="toggleTaskStatus(task)"
                                    v-tooltip="task.completed ? 'Đánh dấu chưa hoàn thành' : 'Đánh dấu hoàn thành'" />

                                <Button icon="pi pi-pencil" class="p-button-rounded p-button-text edit-button"
                                    @click="startEditing(task)" v-tooltip="'Chỉnh sửa'" />
                                <Button icon="pi pi-trash" class="p-button-rounded p-button-text delete-button"
                                    @click="deleteTask(task.id)" v-tooltip="'Xóa'" />
                            </div>
                        </li>
                    </TransitionGroup>

                    <!-- Hiển thị khi không có công việc -->
                    <div v-if="filteredAndSortedTasks.length === 0" class="empty-state">
                        <i class="pi pi-inbox empty-icon"></i>
                        <p v-if="tasks.length === 0">Bạn chưa có công việc nào. Hãy thêm công việc mới!</p>
                        <p v-else>Không tìm thấy công việc phù hợp với bộ lọc hiện tại.</p>
                    </div>
                </div>

                <!-- Tổng kết -->
                <div class="task-summary" v-if="tasks.length > 0">
                    <Button label="Xóa đã hoàn thành" icon="pi pi-trash" class="clear-button p-button-outlined"
                        @click="clearCompleted" v-if="completedTasks > 0" />
                </div>
            </template>
        </Card>
    </div>
</template>

<style scoped>
.todo-app {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    padding: 20px;
}

.todo-app.dark-theme {
    background: linear-gradient(135deg, #2c3e50 0%, #1a1a2e 100%);
}

.todo-card {
    width: 100%;
    max-width: 650px;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
    border-radius: 12px;
    overflow: hidden;
}

.dark-theme .todo-card {
    background-color: #1e1e2d;
    color: #e0e0e0;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.3);
}

.todo-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px;
    background: linear-gradient(to right, #4f46e5, #7e3af2);
    color: white;
}

.todo-title {
    margin: 0;
    font-size: 2.2rem;
    font-weight: 700;
    background: linear-gradient(to right, #ffffff, #e0e0e0);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.dark-mode-toggle {
    color: white;
}

.add-task-container {
    display: flex;
    margin-bottom: 20px;
    gap: 10px;
}

.task-input {
    border-radius: 8px;
    padding: 12px;
    font-size: 1rem;
    transition: box-shadow 0.3s ease;
}

.task-input:focus {
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.2);
}

.add-button {
    background: linear-gradient(to right, #4f46e5, #7e3af2);
    border: none;
    box-shadow: 0 4px 6px rgba(79, 70, 229, 0.2);
    transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.add-button:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 6px 8px rgba(79, 70, 229, 0.3);
}

.add-button:disabled {
    background: #d1d5db;
    box-shadow: none;
}

.search-filter-container {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-bottom: 20px;
}

.search-box {
    flex: 1;
    min-width: 200px;
}

.filter-sort-section {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 10px;
}

.priority-filter,
.sort-dropdown {
    min-width: 120px;
}

.filter-container {
    margin-bottom: 20px;
}

.filter-buttons {
    width: 100%;
    border-radius: 8px;
    overflow: hidden;
}

.progress-section {
    margin-bottom: 20px;
}

.progress-stats {
    display: flex;
    justify-content: space-between;
    margin-bottom: 8px;
    font-size: 0.9rem;
    color: #6b7280;
}

.dark-theme .progress-stats {
    color: #a0aec0;
}

.progress-bar {
    height: 8px;
    border-radius: 4px;
}

/* Styles for batch actions */
.batch-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
    padding: 10px;
    background-color: #f9fafb;
    border-radius: 8px;
    border: 1px solid #e5e7eb;
}

.dark-theme .batch-actions {
    background-color: #2d3748;
    border-color: #4a5568;
}

.batch-actions-left {
    display: flex;
    align-items: center;
}

.select-all-label {
    margin-left: 8px;
    font-size: 0.9rem;
    color: #4b5563;
}

.dark-theme .select-all-label {
    color: #d1d5db;
}

.batch-actions-right {
    display: flex;
    gap: 8px;
}

.select-task-checkbox {
    margin-right: 10px;
}

.tasks-container {
    margin-bottom: 20px;
    max-height: 50vh;
    overflow-y: auto;
    padding-right: 5px;
}

.tasks-container::-webkit-scrollbar {
    width: 6px;
}

.tasks-container::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-radius: 3px;
}

.dark-theme .tasks-container::-webkit-scrollbar-track {
    background: #2d3748;
}

.tasks-container::-webkit-scrollbar-thumb {
    background: #cbd5e0;
    border-radius: 3px;
}

.dark-theme .tasks-container::-webkit-scrollbar-thumb {
    background: #4a5568;
}

.tasks-container::-webkit-scrollbar-thumb:hover {
    background: #a0aec0;
}

.task-list {
    list-style-type: none;
    padding: 0;
    margin: 0;
}

.task-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 15px;
    background-color: white;
    border-radius: 10px;
    margin-bottom: 12px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
    transition: all 0.3s ease;
    border-left: 4px solid #e5e7eb;
}

.dark-theme .task-item {
    background-color: #2d3748;
    border-left-color: #4a5568;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.task-item:hover {
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
    transform: translateY(-2px);
}

.dark-theme .task-item:hover {
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.task-item.priority-high {
    border-left-color: #ef4444;
}

.task-item.priority-medium {
    border-left-color: #f59e0b;
}

.task-item.priority-low {
    border-left-color: #10b981;
}

.task-item.completed-task {
    opacity: 0.75;
    background-color: #f9fafb;
}

.dark-theme .task-item.completed-task {
    background-color: #1a202c;
}

.task-content {
    display: flex;
    align-items: center;
    flex: 1;
    min-width: 0;
}

.task-checkbox {
    margin-right: 15px;
}

.task-text-container {
    display: flex;
    flex-direction: column;
    min-width: 0;
}

.task-text {
    font-size: 1rem;
    cursor: pointer;
    transition: color 0.3s ease, text-decoration 0.3s ease;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.dark-theme .task-text {
    color: #e2e8f0;
}

.task-text.completed {
    text-decoration: line-through;
    color: #9ca3af;
}

.task-date {
    color: #9ca3af;
    font-size: 0.75rem;
    margin-top: 4px;
}

.dark-theme .task-date {
    color: #718096;
}

.edit-container {
    display: flex;
    flex: 1;
    gap: 10px;
}

.edit-input {
    flex: 1;
}

.edit-actions {
    display: flex;
    gap: 5px;
}

.task-actions {
    display: flex;
    gap: 5px;
}

.priority-button,
.edit-button,
.delete-button,
.complete-button {
    width: 2.5rem;
    height: 2.5rem;
}

.priority-button {
    color: #f59e0b;
}

.edit-button {
    color: #3b82f6;
}

.delete-button {
    color: #ef4444;
}

.complete-button {
    color: #10b981;
}

.dark-theme .priority-button {
    color: #fbbf24;
}

.dark-theme .edit-button {
    color: #60a5fa;
}

.dark-theme .delete-button {
    color: #f87171;
}

.dark-theme .complete-button {
    color: #34d399;
}

.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px 20px;
    text-align: center;
    color: #6b7280;
}

.dark-theme .empty-state {
    color: #a0aec0;
}

.empty-icon {
    font-size: 3rem;
    margin-bottom: 15px;
    opacity: 0.5;
}

.task-summary {
    display: flex;
    justify-content: center;
    margin-top: 20px;
    padding-top: 15px;
    border-top: 1px solid #e5e7eb;
}

.dark-theme .task-summary {
    border-top-color: #4a5568;
}

.clear-button {
    color: #4f46e5;
    border-color: #4f46e5;
}

.dark-theme .clear-button {
    color: #818cf8;
    border-color: #818cf8;
}

/* Animations */
.list-enter-active,
.list-leave-active {
    transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
}

.list-enter-from {
    opacity: 0;
    transform: translateY(30px);
}

.list-leave-to {
    opacity: 0;
    transform: translateX(-30px);
}

.list-move {
    transition: transform 0.5s ease;
}

@media (max-width: 768px) {
    .search-filter-container {
        flex-direction: column;
    }

    .filter-sort-section {
        width: 100%;
    }

    .priority-filter,
    .sort-dropdown {
        flex: 1;
    }

    .task-actions {
        flex-direction: column;
    }

    .batch-actions {
        flex-direction: column;
        gap: 10px;
    }

    .batch-actions-left,
    .batch-actions-right {
        width: 100%;
        justify-content: space-between;
    }
}
</style>