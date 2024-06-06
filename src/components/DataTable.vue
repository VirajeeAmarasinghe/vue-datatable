<template>
    <div class="container">

        <div v-if="isLoading" class="loader">
            <div class="spinner"></div>
        </div>

        <div v-else>
            <input type="text" v-model="searchQuery" placeholder="Search" @input="debouncedSearch"
                class="search-input" />
            <table class="table">
                <thead>
                    <tr>
                        <th @click="sort('id')" class="sortable sortable-id">ID</th>
                        <th>Name</th>
                        <th @click="sort('email')" class="sortable">Email</th>
                        <th>Body</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="comment in paginatedComments" :key="comment.id">
                        <td class="sortable-id">{{ comment.id }}</td>
                        <td>{{ comment.name }}</td>
                        <td>{{ comment.email }}</td>
                        <td>{{ comment.body }}</td>
                        <td>
                            <button @click="editComment(comment)" class="btn-action">Edit</button>
                            <button @click="removeComment(comment.id)" class="btn-action">Remove</button>
                        </td>
                    </tr>
                </tbody>
            </table>
            <div class="pagination">
                <button @click="prevPage" :disabled="currentPage === 1" class="pagination-button">Previous</button>
                <span class="pagination-info">Page {{ currentPage }} of {{ totalPages }}</span>
                <button @click="nextPage" :disabled="currentPage === totalPages" class="pagination-button">
                    Next
                </button>
            </div>
            <select v-model="rowsPerPage" @change="updatePage" class="rows-per-page-select">
                <option value="10">10 per page</option>
                <option value="15">15 per page</option>
                <option value="20">20 per page</option>
                <option value="0">All</option>
            </select>
            <!-- Edit form -->
            <div v-if="isEditing" class="edit-form">
                <h3>Edit Comment</h3>
                <form @submit.prevent="updateComment">
                    <label>ID: {{ editingComment.id }}</label><br>
                    <label>Name: <input v-model="editingComment.name" required /></label><br>
                    <label>Email: <input v-model="editingComment.email" required /></label><br>
                    <label>Body: <textarea v-model="editingComment.body" required></textarea></label><br>
                    <button type="submit">Save</button>
                    <button @click="cancelEdit">Cancel</button>
                </form>
            </div>
        </div>
    </div>

</template>

<script>
import { ref, computed, onMounted, watch } from "vue";
import axios from "axios";
import debounce from "lodash/debounce";

export default {
    setup() {
        const comments = ref([]);
        const searchQuery = ref("");
        const currentPage = ref(1);
        const rowsPerPage = ref(10);
        const sortOrder = ref("asc");
        const sortColumn = ref("id");
        const isLoading = ref(false);
        const isEditing = ref(false);
        const editingComment = ref(null);


        const fetchComments = async () => {
            isLoading.value = true;
            try {
                const response = await axios.get(
                    "https://jsonplaceholder.typicode.com/comments"
                );
                comments.value = response.data;
            } catch (error) {
                console.error(error);
            } finally {
                isLoading.value = false;
            }
        };

        const sortedComments = computed(() => {
            return [...comments.value].sort((a, b) => {
                const modifier = sortOrder.value === "asc" ? 1 : -1;
                if (a[sortColumn.value] < b[sortColumn.value]) return -1 * modifier;
                if (a[sortColumn.value] > b[sortColumn.value]) return 1 * modifier;
                return 0;
            });
        });

        const filteredComments = computed(() => {
            let filtered = sortedComments.value;
            if (searchQuery.value) {
                filtered = filtered.filter(
                    (comment) =>
                        comment.email
                            .toLowerCase()
                            .includes(searchQuery.value.toLowerCase()) ||
                        comment.body.toLowerCase().includes(searchQuery.value.toLowerCase())
                );
            }

            if (sortColumn.value === 'id') {
                return sortOrder.value === 'asc' ? filtered.sort((a, b) => a.id - b.id) : filtered.sort((a, b) => b.id - a.id);
            } else if (sortColumn.value === 'email') {
                return sortOrder.value === 'asc' ? filtered.sort((a, b) => a.email.localeCompare(b.email)) : filtered.sort((a, b) => b.email.localeCompare(a.email));
            }
            return filtered;

        });

        const totalPages = computed(() => {
            // Return 1 if rowsPerPage is set to 0 (All)
            if (rowsPerPage.value === 0) return 1;
            const totalItems = filteredComments.value.length;
            return Math.ceil(totalItems / rowsPerPage.value);
        });

        const paginatedComments = computed(() => {
            if (parseInt(rowsPerPage.value) === 0) {
                return filteredComments.value;
            }
            const start = parseInt((currentPage.value - 1) * rowsPerPage.value);
            const end = start + parseInt(rowsPerPage.value);
            return filteredComments.value.slice(start, end);
        });

        const prevPage = () => {
            if (currentPage.value > 1) currentPage.value--;
        };

        const nextPage = () => {
            const nextPageNumber = currentPage.value + 1;
            if (nextPageNumber <= totalPages.value) {
                currentPage.value = nextPageNumber;
            }
        };

        const updatePage = () => {
            currentPage.value = 1;
        };

        // Re-fetch comments on perPage change
        watch(rowsPerPage, updatePage);

        const sort = (column) => {
            if (sortColumn.value === column) {
                sortOrder.value = sortOrder.value === "asc" ? "desc" : "asc";
            } else {
                sortColumn.value = column;
                sortOrder.value = "asc";
            }
            currentPage.value = 1;
        };

        const debouncedSearch = debounce(() => {
            currentPage.value = 1;
        }, 300);

        onMounted(fetchComments);

        const editComment = (comment) => {
            isEditing.value = true;
            editingComment.value = { ...comment };
        };

        const cancelEdit = () => {
            isEditing.value = false;
            editingComment.value = null;
        };

        const updateComment = () => {
            const index = comments.value.findIndex((c) => c.id === editingComment.value.id);
            if (index !== -1) {
                comments.value.splice(index, 1, { ...editingComment.value });
            }
            cancelEdit();
        };

        const removeComment = (id) => {
            const index = comments.value.findIndex((comment) => comment.id === id);
            if (index !== -1) {
                comments.value.splice(index, 1);
            }
        };

        return {
            comments,
            searchQuery,
            currentPage,
            rowsPerPage,
            paginatedComments,
            totalPages,
            prevPage,
            nextPage,
            sort,
            debouncedSearch,
            isLoading,
            isEditing,
            editingComment,
            editComment,
            cancelEdit,
            updateComment,
            removeComment,
        };
    },
};
</script>

<style scoped>
.container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
}

.loader {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100px;
}

.spinner {
    width: 50px;
    height: 50px;
    border: 5px solid #3498db;
    border-radius: 50%;
    border-top-color: transparent;
    animation: spin 1s linear infinite;
}

.table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 4%;
}

th,
td {
    border: 1px solid #ddd;
    padding: 12px;
    text-align: left;
}

th {
    background-color: #f2f2f2;
}

.sortable {
    cursor: pointer;
    color: #3498db;
    position: relative;
}

.sortable:hover {
    background-color: #e6f7ff;
}

.sortable::after {
    content: '';
    position: absolute;
    right: 10px;
    border: 6px solid transparent;
    border-top-color: #3498db;
}

th.sorted-asc::after {
    border-bottom-color: #3498db;
    border-top-color: transparent;
}

th.sorted-desc::after {
    border-top-color: #3498db;
}

.pagination {
    margin-top: 20px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.pagination-button {
    padding: 10px 20px;
    margin: 0 5px;
    background-color: #3498db;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.pagination-button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
}

.pagination-info {
    margin: 0 10px;
}

.rows-per-page-select {
    margin-top: 10px;
    padding: 8px;
    border-radius: 5px;
}

.search-input {
    margin-bottom: 10px;
    padding: 10px;
    border-radius: 5px;
    border: 1px solid #ddd;
    width: 100%;
    box-sizing: border-box;
}

.spinner-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100px;
}

.sortable-id {
    width: 8%;
}

.edit-form {
    margin-top: 20px;
    padding: 20px;
    border: 1px solid #ddd;
    border-radius: 5px;
    background-color: #f9f9f9;
}

.edit-form h3 {
    margin-bottom: 10px;
}

.edit-form label {
    display: block;
    margin-bottom: 10px;
}

.edit-form input,
.edit-form textarea {
    width: 100%;
    padding: 8px;
    margin-top: 5px;
    border: 1px solid #ddd;
    border-radius: 5px;
}

.edit-form button {
    padding: 10px 20px;
    margin-right: 10px;
    background-color: #3498db;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.edit-form button:last-child {
    background-color: #cccccc;
}

.btn-action {
    width: 100%;
    margin-bottom: 4%;
}

@keyframes spin {
    from {
        transform: rotate(0deg);
    }

    to {
        transform: rotate(360deg);
    }
}
</style>
