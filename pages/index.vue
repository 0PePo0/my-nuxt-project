<template>
  <div class="products-table-container">

    <!-- Loading state -->
    <div v-if="loading" class="loading-state">
      <div class="spinner"></div>
      <p>Завантаження даних...</p>
    </div>

    <!-- Error state -->
    <div v-else-if="error" class="error-state">
      <p>Помилка завантаження даних: {{ error }}</p>
      <button @click="fetchProducts" class="retry-button">Спробувати знову</button>
    </div>

    <div v-else class="table-container">

      <!-- Table -->
      <div class="table-wrapper">
        <table class="data-table">
          <thead>
          <tr>
            <th
                v-for="column in columns"
                :key="column.key"
                class="table-header"
                @click="sortTable(column.key)"
            >
              <div class="header-content">
                {{ column.label }}
                <span v-if="column.sortable" class="sort-icon">
                    <template v-if="sort.column === column.key">
                      <svg v-if="sort.direction === 'asc'" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="m18 15-6-6-6 6"/>
                      </svg>
                      <svg v-else xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="m6 9 6 6 6-6"/>
                      </svg>
                    </template>
                    <svg v-else xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="inactive-sort">
                      <path d="m7 15 5 5 5-5"/>
                      <path d="m7 9 5-5 5 5"/>
                    </svg>
                  </span>
              </div>
            </th>
          </tr>
          </thead>
          <tbody>
          <tr v-for="product in paginatedData" :key="product.id" class="table-row">
            <td class="table-cell">{{ product.title }}</td>
            <td class="table-cell description-cell">{{ product.description }}</td>
            <td class="table-cell">{{ formatPrice(product.price) }}</td>
            <td class="table-cell">
                <span :class="getRatingClass(product.rating)">
                  {{ product.rating }}
                </span>
            </td>
            <td class="table-cell">{{ product.brand }}</td>
            <td class="table-cell">{{ product.category }}</td>
            <td class="table-cell thumbnail-cell">
              <img :src="product.thumbnail" :alt="product.title" class="product-thumbnail" />
            </td>
          </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div class="pagination-container">
        <div class="pagination-info">
          Показано {{ startIndex + 1 }} - {{ Math.min(endIndex, filteredData.length) }} з {{ filteredData.length }} товарів
        </div>
        <div class="pagination-controls">
          <button
              @click="page = Math.max(1, page - 1)"
              :disabled="page === 1"
              class="pagination-button"
              :class="{ 'disabled': page === 1 }"
          >
            Попередня
          </button>
          <div class="pagination-pages">
            <button
                v-for="p in displayedPages"
                :key="p"
                @click="page = p"
                class="page-button"
                :class="{ 'active': page === p }"
            >
              {{ p }}
            </button>
          </div>
          <button
              @click="page = Math.min(totalPages, page + 1)"
              :disabled="page === totalPages"
              class="pagination-button"
              :class="{ 'disabled': page === totalPages }"
          >
            Наступна
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

// API URL
const API_URL = 'https://dummyjson.com/products'

// State variables
const products = ref([])
const loading = ref(true)
const error = ref(null)

// Table columns configuration
const columns = [
  { key: 'title', label: 'Назва', sortable: true },
  { key: 'description', label: 'Опис', sortable: true },
  { key: 'price', label: 'Ціна', sortable: true },
  { key: 'rating', label: 'Оцінка', sortable: true },
  { key: 'brand', label: 'Бренд', sortable: true },
  { key: 'category', label: 'Категорія', sortable: true },
  { key: 'thumbnail', label: 'Фото', sortable: false },
]
// Pagination state
const page = ref(1)
const perPage = ref(5)
const startIndex = computed(() => (page.value - 1) * perPage.value)
const endIndex = computed(() => startIndex.value + perPage.value)
const totalPages = computed(() => Math.ceil(filteredData.value.length / perPage.value))

// Calculate which page numbers to display
const displayedPages = computed(() => {
  const pages = []
  const maxVisiblePages = 5

  if (totalPages.value <= maxVisiblePages) {
    // Show all pages if there are few
    for (let i = 1; i <= totalPages.value; i++) {
      pages.push(i)
    }
  } else {
    // Always show first page
    pages.push(1)

    // Calculate middle pages
    let startPage = Math.max(2, page.value - 1)
    let endPage = Math.min(totalPages.value - 1, page.value + 1)

    // Adjust if at the beginning or end
    if (page.value <= 2) {
      endPage = 3
    } else if (page.value >= totalPages.value - 1) {
      startPage = totalPages.value - 2
    }

    // Add ellipsis if needed
    if (startPage > 2) {
      pages.push('...')
    }

    // Add middle pages
    for (let i = startPage; i <= endPage; i++) {
      pages.push(i)
    }

    // Add ellipsis if needed
    if (endPage < totalPages.value - 1) {
      pages.push('...')
    }

    // Always show last page
    pages.push(totalPages.value)
  }

  return pages
})

// Search state
const searchQuery = ref('')

// Sorting state
const sort = ref({ column: 'title', direction: 'asc' })

// Filter data based on search query
const filteredData = computed(() => {
  if (!searchQuery.value) return products.value

  const query = searchQuery.value.toLowerCase()
  return products.value.filter(product => {
    return (
        product.title.toLowerCase().includes(query) ||
        product.description.toLowerCase().includes(query) ||
        product.brand.toLowerCase().includes(query) ||
        product.category.toLowerCase().includes(query)
    )
  })
})

// Sort the filtered data
const sortedData = computed(() => {
  const { column, direction } = sort.value

  if (!column) return filteredData.value

  return [...filteredData.value].sort((a, b) => {
    let aValue = a[column]
    let bValue = b[column]

    // Handle special case for price (ensure it's treated as a number)
    if (column === 'price') {
      aValue = Number(aValue)
      bValue = Number(bValue)
    }

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

// Format price with currency
function formatPrice(price) {
  return new Intl.NumberFormat('uk-UA', {
    style: 'currency',
    currency: 'UAH',
    minimumFractionDigits: 2
  }).format(price)
}

// Get class for rating based on value
function getRatingClass(rating) {
  return rating < 4.5 ? 'rating-low' : 'rating-high'
}

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

// Fetch products from API
async function fetchProducts() {
  loading.value = true
  error.value = null

  try {
    const response = await fetch(API_URL)

    const data = await response.json()
    products.value = data.products || []
  } catch (err) {
    error.value = err.message
    console.error('Error fetching products:', err)
  } finally {
    loading.value = false
  }
}

// Fetch data on component mount
onMounted(() => {
  fetchProducts()
})
</script>

<style scoped>
/* Container styles */
.products-table-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

/* Loading state */
.loading-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(0, 0, 0, 0.1);
  border-radius: 50%;
  border-top-color: #3b82f6;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Error state */
.error-state {
  padding: 24px;
  text-align: center;
  background-color: #fee2e2;
  border-radius: 8px;
  color: #991b1b;
}

.retry-button {
  margin-top: 16px;
  padding: 8px 16px;
  background-color: #ef4444;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
}

.retry-button:hover {
  background-color: #dc2626;
}

/* Table styles */
.table-container {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.table-wrapper {
  overflow-x: auto;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  background-color: white;
}

.table-header {
  padding: 12px 16px;
  text-align: left;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  color: #64748b;
  background-color: #f8fafc;
  cursor: pointer;
  user-select: none;
  border-bottom: 1px solid #e2e8f0;
}

.header-content {
  display: flex;
  align-items: center;
}

.sort-icon {
  margin-left: 4px;
}

.inactive-sort {
  color: #cbd5e1;
}

.table-row {
  border-bottom: 1px solid #e2e8f0;
  transition: background-color 0.2s;
}

.table-row:hover {
  background-color: #f8fafc;
}

.table-cell {
  padding: 12px 16px;
  vertical-align: middle;
}

/* Description cell with truncation */
.description-cell {
  max-width: 300px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}



/* Thumbnail styles */
.thumbnail-cell {
  padding: 8px;
}

.product-thumbnail {
  width: 100px;
  height: 100px;
  object-fit: cover;
  border-radius: 4px;
  border: 1px solid #e2e8f0;
}

/* Pagination styles */
.pagination-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 16px;
  flex-wrap: wrap;
  gap: 16px;
}

.pagination-info {
  font-size: 14px;
  color: #64748b;
}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.pagination-pages {
  display: flex;
  gap: 4px;
}

.pagination-button, .page-button {
  padding: 6px 12px;
  border: 1px solid #e2e8f0;
  background-color: white;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s, border-color 0.2s;
}

.pagination-button:hover:not(.disabled), .page-button:hover:not(.active) {
  background-color: #f8fafc;
}

.page-button {
  min-width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}

.page-button.active {
  background-color: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.pagination-button.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .pagination-container {
    flex-direction: column;
    align-items: flex-start;
  }

  .table-cell, .table-header {
    padding: 8px 12px;
  }

  .description-cell {
    max-width: 150px;
  }
}
</style>