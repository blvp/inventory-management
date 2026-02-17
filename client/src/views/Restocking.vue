<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-based restocking recommendations prioritized by demand trend</p>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget Controls</h3>
      </div>
      <div class="budget-controls">
        <div class="budget-slider-row">
          <label class="budget-label" for="budget-slider">Budget</label>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          id="budget-slider"
          v-model.number="budget"
          type="range"
          min="5000"
          max="200000"
          step="1000"
          class="budget-slider"
        />
        <div class="budget-range-labels">
          <span>$5,000</span>
          <span>$200,000</span>
        </div>
        <div class="budget-actions">
          <button
            class="btn-primary"
            :disabled="loading"
            @click="loadRecommendations"
          >
            {{ loading ? 'Loading...' : 'Get Recommendations' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="error" class="error">{{ error }}</div>

    <div v-if="successMessage" class="success-message">{{ successMessage }}</div>

    <template v-if="recommendations.length > 0">
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Total Cost</div>
          <div class="stat-value">{{ formatCurrency(totalCost) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Budget Remaining</div>
          <div class="stat-value">{{ formatCurrency(budgetRemaining) }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Items Selected</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Forecasted Demand</th>
                <th>On Hand</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Restock Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ item.restock_qty.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td><strong>${{ item.restock_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="table-footer">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </template>

    <div v-else-if="!loading && hasSearched" class="card">
      <div class="loading">No recommendations available for the selected budget. Try increasing the budget amount.</div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const recommendations = ref([])
    const loading = ref(false)
    const submitting = ref(false)
    const error = ref(null)
    const successMessage = ref(null)
    const hasSearched = ref(false)
    let successTimer = null

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.restock_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    // Clear stale recommendations when budget changes so summary cards stay accurate
    watch(budget, () => {
      if (recommendations.value.length > 0) {
        recommendations.value = []
        hasSearched.value = false
      }
    })

    const formatCurrency = (value) =>
      '$' + value.toLocaleString(undefined, { minimumFractionDigits: 0, maximumFractionDigits: 0 })

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      successMessage.value = null
      hasSearched.value = true

      if (successTimer) {
        clearTimeout(successTimer)
        successTimer = null
      }

      try {
        recommendations.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + (err.response?.data?.detail || err.message)
        recommendations.value = []
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      submitting.value = true
      error.value = null

      const orderPayload = {
        items: recommendations.value.map(item => ({
          sku: item.sku,
          item_name: item.item_name,
          quantity: item.restock_qty,
          unit_cost: item.unit_cost,
          line_cost: item.restock_cost
        })),
        total_cost: totalCost.value
      }

      try {
        const result = await api.submitRestockingOrder(orderPayload)
        const orderNumber = result.order_number || result.id || 'RST-' + Date.now()
        successMessage.value = `Order placed successfully! Order #${orderNumber} has been submitted.`
        recommendations.value = []
        hasSearched.value = false

        if (successTimer) clearTimeout(successTimer)
        successTimer = setTimeout(() => {
          successMessage.value = null
          successTimer = null
        }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    return {
      budget,
      recommendations,
      loading,
      submitting,
      error,
      successMessage,
      hasSearched,
      totalCost,
      budgetRemaining,
      formatCurrency,
      loadRecommendations,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-controls {
  padding: 0.5rem 0;
}

.budget-slider-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  transform: scale(1.15);
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border: none;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1.25rem;
}

.budget-actions {
  display: flex;
  justify-content: flex-end;
}

.btn-primary {
  display: inline-flex;
  align-items: center;
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, box-shadow 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.table-footer {
  display: flex;
  justify-content: flex-end;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
