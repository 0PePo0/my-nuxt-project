<template>
  <div class="p-6 max-w-6xl mx-auto">
    <h1 class="text-2xl font-bold mb-6">Data Table</h1>

    <!-- Search input -->
    <div class="mb-4">
      <div class="relative max-w-md">
        <input
            v-model="searchQuery"
            type="text"
            placeholder="Search..."
            class="w-full px-4 py-2 border rounded-md pl-10 focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        <div class="absolute left-3 top-1/2 transform -translate-y-1/2">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-gray-400">
            <circle cx="11" cy="11" r="8"></circle>
            <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
          </svg>
        </div>
      </div>
    </div>

    <!-- Table component -->
    <div class="overflow-x-auto">
      <table class="min-w-full bg-white border border-gray-200 rounded-lg">
        <thead>
        <tr class="bg-gray-100">
          <th
              v-for="column in columns"
              :key="column.key"
              class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer"
              @click="sortTable(column.key)"
          >
            <div class="flex items-center">
              {{ column.label }}
              <span v-if="column.sortable" class="ml-1">
                  <template v-if="sort.column === column.key">
                    <svg v-if="sort.direction === 'asc'" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="m18 15-6-6-6 6"/>
                    </svg>
                    <svg v-else xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="m6 9 6 6 6-6"/>
                    </svg>
                  </template>
                  <svg v-else xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-gray-300">
                    <path d="m7 15 5 5 5-5"/>
                    <path d="m7 9 5-5 5 5"/>
                  </svg>
                </span>
            </div>
          </th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="row in paginatedData" :key="row.id" class="border-t border-gray-200 hover:bg-gray-50">
          <td v-for="column in columns" :key="column.key" class="px-6 py-4 whitespace-nowrap">
            <template v-if="column.key === 'status'">
                <span
                    :class="[
                    'px-2 py-1 text-xs font-medium rounded-full',
                    row.status === 'Active' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
                  ]"
                >
                  {{ row.status }}
                </span>
            </template>
            <template v-else>
              {{ row[column.key] }}
            </template>
          </td>
        </tr>
        <tr v-if="paginatedData.length === 0">
          <td :colspan="columns.length" class="px-6 py-10 text-center text-gray-500">
            <div class="flex flex-col items-center justify-center">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-gray-400 mb-2">
                <path d="M4 22h14a2 2 0 0 0 2-2V7.5L14.5 2H6a2 2 0 0 0-2 2v4"></path>
                <path d="M14 2v6h6"></path>
                <path d="M3 15h6"></path>
                <path d="M9 18H3"></path>
              </svg>
              <p>No data found</p>
            </div>
          </td>
        </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="mt-4 flex items-center justify-between">
      <div class="text-sm text-gray-500">
        Showing {{ startIndex + 1 }} to {{ Math.min(endIndex, filteredData.length) }} of {{ filteredData.length }} entries
      </div>
      <div class="flex items-center space-x-2">
        <button
            @click="page = Math.max(1, page - 1)"
            :disabled="page === 1"
            class="px-3 py-1 border rounded-md text-sm disabled:opacity-50"
            :class="page === 1 ? 'cursor-not-allowed' : 'hover:bg-gray-50'"
        >
          Previous
        </button>
        <div class="flex space-x-1">
          <button
              v-for="p in totalPages"
              :key="p"
              @click="page = p"
              class="w-8 h-8 flex items-center justify-center rounded-md text-sm"
              :class="page === p ? 'bg-blue-500 text-white' : 'border hover:bg-gray-50'"
          >
            {{ p }}
          </button>
        </div>
        <button
            @click="page = Math.min(totalPages, page + 1)"
            :disabled="page === totalPages"
            class="px-3 py-1 border rounded-md text-sm disabled:opacity-50"
            :class="page === totalPages ? 'cursor-not-allowed' : 'hover:bg-gray-50'"
        >
          Next
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

// Sample data
const data = ref([
  { id: 1, name: 'John Doe', email: 'john@example.com', role: 'Admin', status: 'Active', lastLogin: '2023-10-15' },
  { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'User', status: 'Active', lastLogin: '2023-10-14' },
  { id: 3, name: 'Bob Johnson', email: 'bob@example.com', role: 'Editor', status: 'Inactive', lastLogin: '2023-09-20' },
  { id: 4, name: 'Alice Brown', email: 'alice@example.com', role: 'User', status: 'Active', lastLogin: '2023-10-12' },
  { id: 5, name: 'Charlie Wilson', email: 'charlie@example.com', role: 'Admin', status: 'Active', lastLogin: '2023-10-10' },
  { id: 6, name: 'Diana Miller', email: 'diana@example.com', role: 'User', status: 'Inactive', lastLogin: '2023-08-05' },
  { id: 7, name: 'Edward Davis', email: 'edward@example.com', role: 'Editor', status: 'Active', lastLogin: '2023-10-08' },
  { id: 8, name: 'Fiona Clark', email: 'fiona@example.com', role: 'User', status: 'Active', lastLogin: '2023-10-05' },
  { id: 9, name: 'George White', email: 'george@example.com', role: 'Admin', status: 'Inactive', lastLogin: '2023-09-15' },
  { id: 10, name: 'Hannah Green', email: 'hannah@example.com', role: 'User', status: 'Active', lastLogin: '2023-10-01' },
  { id: 11, name: 'Ian Black', email: 'ian@example.com', role: 'Editor', status: 'Active', lastLogin: '2023-09-28' },
  { id: 12, name: 'Julia Red', email: 'julia@example.com', role: 'User', status: 'Inactive', lastLogin: '2023-08-20' },
  { id: 13, name: 'Kevin Gray', email: 'kevin@example.com', role: 'Admin', status: 'Active', lastLogin: '2023-09-25' },
  { id: 14, name: 'Laura Blue', email: 'laura@example.com', role: 'User', status: 'Active', lastLogin: '2023-09-22' },
  { id: 15, name: 'Mike Orange', email: 'mike@example.com', role: 'Editor', status: 'Inactive', lastLogin: '2023-07-15' },
])

// Table columns configuration
const columns = [
  { key: 'id', label: 'ID', sortable: true },
  { key: 'name', label: 'Name', sortable: true },
  { key: 'email', label: 'Email', sortable: true },
  { key: 'role', label: 'Role', sortable: true },
  { key: 'status', label: 'Status', sortable: true },
  { key: 'lastLogin', label: 'Last Login', sortable: true },
]

// Pagination state
const page = ref(1)
const perPage = ref(5)
const startIndex = computed(() => (page.value - 1) * perPage.value)
const endIndex = computed(() => startIndex.value + perPage.value)
const totalPages = computed(() => Math.ceil(filteredData.value.length / perPage.value))

// Search state
const searchQuery = ref('')

// Sorting state
const sort = ref({ column: 'id', direction: 'asc' })

// Filter data based on search query
const filteredData = computed(() => {
  if (!searchQuery.value) return data.value

  const query = searchQuery.value.toLowerCase()
  return data.value.filter(item => {
    return Object.values(item).some(value =>
        String(value).toLowerCase().includes(query)
    )
  })
})

// Sort the filtered data
const sortedData = computed(() => {
  const { column, direction } = sort.value

  if (!column) return filteredData.value

  return [...filteredData.value].sort((a, b) => {
    const aValue = a[column]
    const bValue = b[column]

    if (typeof aValue === 'string') {
      return direction === 'asc'
          ? aValue.localeCompare(bValue)
          : bValue.localeCompare(aValue)
    } else {
      return direction === 'asc'
          ? aValue - bValue
          : bValue - aValue
    }
  })
})

// Get paginated data from sorted data
const paginatedData = computed(() => {
  return sortedData.value.slice(startIndex.value, endIndex.value)
})

// Reset to first page when search query changes
watch(searchQuery, () => {
  page.value = 1
})

// Handle sorting
function sortTable(column) {
  if (sort.value.column === column) {
    // Toggle direction if clicking the same column
    sort.value.direction = sort.value.direction === 'asc' ? 'desc' : 'asc'
  } else {
    // Set new column with default ascending direction
    sort.value = { column, direction: 'asc' }
  }
}
</script>

<style scoped>
/* Add any custom styles here */
</style>