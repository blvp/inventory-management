<template>
  <div class="backlog">
    <div class="page-header">
      <h2>{{ t('backlog.title') }}</h2>
      <p>{{ t('backlog.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card danger">
          <div class="stat-label">{{ t('backlog.highPriority') }}</div>
          <div class="stat-value">{{ highPriorityItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('backlog.mediumPriority') }}</div>
          <div class="stat-value">{{ mediumPriorityItems.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('backlog.lowPriority') }}</div>
          <div class="stat-value">{{ lowPriorityItems.length }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('backlog.totalItems') }}</div>
          <div class="stat-value">{{ backlogItems.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('backlog.backlogItems') }}</h3>
        </div>
        <div v-if="backlogItems.length === 0" style="padding: 3rem; text-align: center;">
          <p style="font-size: 1.125rem; color: #10b981; font-weight: 600;">
            {{ t('backlog.noItems') }}
          </p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('backlog.table.orderId') }}</th>
                <th>{{ t('backlog.table.sku') }}</th>
                <th>{{ t('backlog.table.itemName') }}</th>
                <th>{{ t('backlog.table.quantityNeeded') }}</th>
                <th>{{ t('backlog.table.quantityAvailable') }}</th>
                <th>{{ t('backlog.table.shortage') }}</th>
                <th>{{ t('backlog.table.daysDelayed') }}</th>
                <th>{{ t('backlog.table.priority') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in backlogItems" :key="item.id">
                <td><strong>{{ item.order_id }}</strong></td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.quantity_needed }}</td>
                <td>{{ item.quantity_available }}</td>
                <td>
                  <span class="badge danger">
                    {{ item.quantity_needed - item.quantity_available }} {{ t('backlog.unitsShort') }}
                  </span>
                </td>
                <td>
                  <span :style="{ color: item.days_delayed > 7 ? '#ef4444' : '#f59e0b' }">
                    {{ item.days_delayed }} {{ t('backlog.days') }}
                  </span>
                </td>
                <td>
                  <span :class="['badge', item.priority]">
                    {{ translatePriority(item.priority) }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Backlog',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const allBacklogItems = ref([])
    const inventoryItems = ref([])

    const { t, translatePriority } = useI18n()

    // Use shared filters
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    // Filter backlog based on inventory filters
    const backlogItems = computed(() => {
      if (selectedLocation.value === 'all' && selectedCategory.value === 'all') {
        return allBacklogItems.value
      }

      // Get SKUs of items that match the filters
      const validSkus = new Set(inventoryItems.value.map(item => item.sku))
      return allBacklogItems.value.filter(b => validSkus.has(b.item_sku))
    })

    const highPriorityItems = computed(() =>
      backlogItems.value.filter(item => item.priority === 'high')
    )
    const mediumPriorityItems = computed(() =>
      backlogItems.value.filter(item => item.priority === 'medium')
    )
    const lowPriorityItems = computed(() =>
      backlogItems.value.filter(item => item.priority === 'low')
    )

    const loadBacklog = async () => {
      try {
        loading.value = true
        const filters = getCurrentFilters()

        const [backlogData, inventoryData] = await Promise.all([
          api.getBacklog(),
          api.getInventory({
            warehouse: filters.warehouse,
            category: filters.category
          })
        ])

        allBacklogItems.value = backlogData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load backlog: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Watch for filter changes and reload data
    watch([selectedLocation, selectedCategory], () => {
      loadBacklog()
    })

    onMounted(loadBacklog)

    return {
      loading,
      error,
      backlogItems,
      highPriorityItems,
      mediumPriorityItems,
      lowPriorityItems,
      t,
      translatePriority
    }
  }
}
</script>

<style scoped>
.backlog {
  padding: 0;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 1.125rem;
  font-weight: 600;
}
</style>
