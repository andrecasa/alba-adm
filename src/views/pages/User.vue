<script setup>
import { FilterMatchMode } from '@primevue/core/api';
import axios from 'axios';
import { useToast } from 'primevue/usetoast';
import { onMounted, ref } from 'vue';
const API_URL = import.meta.env.VITE_API_URL;

const toast = useToast();
const dt = ref();
const users = ref();
const userDialog = ref(false);
const deleteUserDialog = ref(false);
const deleteUsersDialog = ref(false);
const user = ref({});
const selectedUsers = ref([]);
const filters = ref({
    global: { value: null, matchMode: FilterMatchMode.CONTAINS }
});
const submitted = ref(false);
const showPassword = ref(false);

// Helper to get token from cookie
function getTokenFromCookie() {
    const match = document.cookie.match(new RegExp('(^| )token=([^;]+)'));
    return match ? match[2] : null;
}

onMounted(async () => {
    try {
        const token = getTokenFromCookie();
        const res = await axios.get(`${API_URL}/users/all/`, {
            headers: {
                Authorization: `Bearer ${token}`
            }
        });
        return users.value = res.data;
    } catch (e) {
        users.value = [];
        toast.add({ severity: 'error', summary: 'Error', detail: 'Failed to fetch users', life: 3000 });
    }
});

function openNew() {
    user.value = { user_active: true };
    submitted.value = false;
    userDialog.value = true;
}

function hideDialog() {
    userDialog.value = false;
    submitted.value = false;
}

function saveUser() {
    submitted.value = true;

    if (
        !user.value.user_name?.trim() ||
        !user.value.user_email?.trim() ||
        !isValidEmail(user.value.user_email) ||
        (!user.value.user_id && !user.value.user_password?.trim()) 
    ) {
        return; 
    }

    const token = getTokenFromCookie();

    if (user.value.user_id) {
        axios.put(`${API_URL}/users/${user.value.user_id}`, user.value, {
            headers: { Authorization: `Bearer ${token}` }
        })
        .then(res => {
            const idx = users.value.findIndex(u => u.user_id === user.value.user_id);
            if (idx !== -1) users.value[idx] = res.data;
            toast.add({ severity: 'success', summary: 'Successful', detail: 'User Updated', life: 3000 });
            userDialog.value = false;
            user.value = {};
        })
        .catch((error) => {
            const detail = error.response.data.error || 'Failed to update user';
            toast.add({ severity: 'error', summary: 'Error', detail, life: 3000 });
        });
    } else {
        axios.post(`${API_URL}/users/`, user.value, {
            headers: { Authorization: `Bearer ${token}` }
        })
        .then(res => {
            users.value.push(res.data);
            toast.add({ severity: 'success', summary: 'Successful', detail: 'User Created', life: 3000 });
            userDialog.value = false;
            user.value = {};
        })
        .catch((error) => {
            const detail = error.response.data.error || 'Failed to create user';
            toast.add({ severity: 'error', summary: 'Error', detail, life: 3000 });
        });
    }
}

function editUser(usr) {
    user.value = { ...usr };
    userDialog.value = true;
}

function confirmDeleteUser(usr) {
    user.value = usr;
    deleteUserDialog.value = true;
}

function deleteUser() {
    const token = getTokenFromCookie();
    axios.delete(`${API_URL}/users/${user.value.user_id}`, {
        headers: { Authorization: `Bearer ${token}` }
    })
    .then(() => {
        users.value = users.value.filter((val) => val.user_id !== user.value.user_id);
        deleteUserDialog.value = false;
        user.value = {};
        toast.add({ severity: 'success', summary: 'Successful', detail: 'User Deleted', life: 3000 });
    })
    .catch(() => {
        toast.add({ severity: 'error', summary: 'Error', detail: 'Failed to delete user', life: 3000 });
    });
}

function isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}


function exportCSV() {
    dt.value.exportCSV();
}

function confirmDeleteSelected() {
    deleteUsersDialog.value = true;
}

function deleteSelectedUsers() {
    const token = getTokenFromCookie();
    const idsToDelete = selectedUsers.value.map(u => u.user_id);
    Promise.all(
        idsToDelete.map(id =>
            axios.delete(`${API_URL}/users/${id}`, {
                headers: { Authorization: `Bearer ${token}` }
            })
        )
    )
    .then(() => {
        users.value = users.value.filter((val) => !idsToDelete.includes(val.user_id));
        deleteUsersDialog.value = false;
        selectedUsers.value = [];
        toast.add({ severity: 'success', summary: 'Successful', detail: 'Users Deleted', life: 3000 });
    })
    .catch(() => {
        toast.add({ severity: 'error', summary: 'Error', detail: 'Failed to delete users', life: 3000 });
    });
}

</script>

<template>
    
    <div>
        <div class="card">
            <Toolbar class="mb-6">
                <template #start>
                    <Button label="New" icon="pi pi-plus" severity="secondary" class="mr-2" @click="openNew" />
                    <Button label="Delete" icon="pi pi-trash" severity="secondary" @click="confirmDeleteSelected" :disabled="!selectedUsers || !selectedUsers.length" />
                </template>

                <template #end>
                    <Button label="Export" icon="pi pi-upload" severity="secondary" @click="exportCSV($event)" />
                </template>
            </Toolbar>

            <DataTable
                ref="dt"
                v-model:selection="selectedUsers"
                :value="users"
                dataKey="user_id"
                :paginator="true"
                :rows="10"
                :filters="filters"
                stripedRows
                paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
                :rowsPerPageOptions="[5, 10, 25]"
                currentPageReportTemplate="Showing {first} to {last} of {totalRecords} Users"
            >
                <template #header>
                    <div class="flex flex-wrap gap-2 items-center justify-between">
                        <h4 class="m-0">Manage Users</h4>
                        <IconField>
                            <InputIcon>
                                <i class="pi pi-search" />
                            </InputIcon>
                            <InputText v-model="filters['global'].value" placeholder="Search..." />
                        </IconField>
                    </div>
                </template>

                <Column selectionMode="multiple":exportable="false"></Column>
                <Column field="user_id" header="Id" sortable></Column>
                <Column field="user_name" header="Nome" sortable></Column>
                <Column field="user_email" header="Name" sortable></Column>
                <Column field="user_active" header="Active" sortable>
                    <template #body="slotProps">
                        <i v-if="slotProps.data.user_active === true" class="pi pi-check-circle text-green-600 ml-2"></i>
                        <i v-if="slotProps.data.user_active === false" class="pi pi-times-circle text-red-600 ml-2"></i>
                    </template>
                </Column>
                
                <Column :exportable="false" style="text-align: right;">
                    <template #body="slotProps">
                        <Button icon="pi pi-pencil" outlined rounded class="mr-2" @click="editUser(slotProps.data)" />
                        <Button icon="pi pi-trash" outlined rounded severity="danger" @click="confirmDeleteUser(slotProps.data)" />
                    </template>
                </Column>
            </DataTable>
        </div>

        <Dialog v-model:visible="userDialog" :style="{ width: '450px' }" header="User Details" :modal="true">
            <div class="flex flex-col gap-6">
                
                <div>
                    <label for="user_name" class="block font-bold mb-3">Name</label>
                    <InputText id="user_name" v-model.trim="user.user_name" required="true" autofocus :invalid="submitted && !user.user_name" fluid />
                    <small v-if="submitted && !user.user_name" class="text-red-500">Name is required.</small>
                </div>
                <div>
                    <label for="user_email" class="block font-bold mb-3">Email</label>
                    <InputText
                        id="user_email"
                        v-model="user.user_email"
                        required="true"
                        :invalid="submitted && (!user.user_email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(user.user_email))"
                        fluid
                    />
                    <small v-if="submitted && !user.user_email" class="text-red-500">Email is required.</small>
                    <small v-else-if="submitted && user.user_email && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(user.user_email)" class="text-red-500">Invalid email format.</small>
                </div>

                <div v-if="!user.user_id">
                    <label for="user_password" class="block font-bold mb-3">Password</label>
                    <div class="flex items-center gap-2">
                        <InputText
                            id="user_password"
                            v-model="user.user_password"
                            :type="showPassword ? 'text' : 'password'"
                            required="true"
                            rows="3"
                            cols="20"
                            fluid
                        />
                        <Button
                            :icon="showPassword ? 'pi pi-eye-slash' : 'pi pi-eye'"
                            @click="showPassword = !showPassword"
                            type="button"
                            text
                            class="p-0"
                            :aria-label="showPassword ? 'Hide password' : 'Show password'"
                        />
                    </div>
                    <small v-if="submitted && !user.user_password" class="text-red-500">Password is required.</small>
                </div>

                <div>
                    <label for="user_active" class="block font-bold mb-3">Active</label>
                    <Checkbox
                        id="user_active"
                        v-model="user.user_active"
                        :binary="true"
                        :true-value="true"
                        :false-value="false"
                    />
                </div>

            </div>

            <template #footer>
                <Button label="Cancel" icon="pi pi-times" text @click="hideDialog" />
                <Button label="Save" icon="pi pi-check" @click="saveUser" />
            </template>
        </Dialog>

        <Dialog v-model:visible="deleteUserDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span v-if="user">Are you sure you want to delete <b>{{ user.user_name }}</b>?</span>
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteUserDialog = false" />
                <Button label="Yes" icon="pi pi-check" severity="danger" @click="deleteUser" />
            </template>
        </Dialog>

        <Dialog v-model:visible="deleteUsersDialog" :style="{ width: '450px' }" header="Confirm" :modal="true">
            <div class="flex items-center gap-4">
                <i class="pi pi-exclamation-triangle !text-3xl" />
                <span v-if="user">Are you sure you want to delete the selected users?</span>
            </div>
            <template #footer>
                <Button label="No" icon="pi pi-times" text @click="deleteUsersDialog = false" />
                <Button label="Yes" icon="pi pi-check" severity="danger" @click="deleteUser" />
            </template>
        </Dialog>
    </div>
</template>
