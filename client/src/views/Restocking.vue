<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.availableBudget') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-slider-block">
            <label class="budget-slider-label">
              {{ formatCurrency(budget, currentCurrency) }}
            </label>
            <input
              type="range"
              min="0"
              max="500000"
              step="5000"
              v-model.number="budget"
              class="budget-slider"
            />
            <div class="slider-range-labels">
              <span>{{ formatCurrency(0, currentCurrency) }}</span>
              <span>{{ formatCurrency(500000, currentCurrency) }}</span>
            </div>
          </div>
          <div class="budget-stats">
            <div class="budget-stat-card">
              <div class="budget-stat-label">{{ t('restocking.recommendedTotal') }}</div>
              <div class="budget-stat-value">{{ formatCurrency(totalCost, currentCurrency) }}</div>
            </div>
            <div class="budget-stat-card" :class="{ 'over-budget': budgetRemaining < 0 }">
              <div class="budget-stat-label">{{ t('restocking.budgetRemaining') }}</div>
              <div class="budget-stat-value">{{ formatCurrency(budgetRemaining, currentCurrency) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="submittedOrderNumber" class="success-banner">
        {{ t('restocking.submittedMessage', { orderNumber: submittedOrderNumber }) }}
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <div>
            <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
            <p class="card-description">{{ t('restocking.recommendationsDescription') }}</p>
          </div>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.subtotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="recommendations.length === 0">
                <td colspan="6" class="empty-state">{{ t('restocking.noItemsFit') }}</td>
              </tr>
              <!-- Use item.id as key per CLAUDE.md — never use index -->
              <tr v-for="item in recommendations" :key="item.id">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', getTrendBadgeClass(item.trend)]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td>{{ item.restock_quantity }}</td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.subtotal, currentCurrency) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="place-order-footer">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

const DEFAULT_BUDGET = 100000

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    // --- Raw state refs ---
    const forecasts = ref([])
    const inventory = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submittedOrderNumber = ref(null)
    const budget = ref(DEFAULT_BUDGET)

    // --- Derived lookup: SKU -> inventory item ---
    const inventoryBySku = computed(() =>
      Object.fromEntries(inventory.value.map(i => [i.sku, i]))
    )

    // --- Candidates: all forecast rows with a positive restock gap and a known price ---
    // restock_quantity = max(0, forecasted_demand - current_demand)
    // Items with zero gap or missing price are excluded (nothing to order / can't value them)
    const candidates = computed(() => {
      return forecasts.value
        .map(f => {
          const restock_quantity = Math.max(0, (f.forecasted_demand ?? 0) - (f.current_demand ?? 0))
          const invItem = inventoryBySku.value[f.item_sku]
          const unit_cost = invItem?.unit_cost ?? 0
          const subtotal = restock_quantity * unit_cost
          return { ...f, restock_quantity, unit_cost, subtotal }
        })
        .filter(c => c.restock_quantity > 0 && c.unit_cost > 0)
        .sort((a, b) => {
          // Increasing trend first, then by subtotal desc, then SKU as tiebreaker
          const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
          const ta = trendOrder[a.trend] ?? 3
          const tb = trendOrder[b.trend] ?? 3
          if (ta !== tb) return ta - tb
          if (b.subtotal !== a.subtotal) return b.subtotal - a.subtotal
          return a.item_sku.localeCompare(b.item_sku)
        })
    })

    // --- Greedy fit: include each candidate whose running total still fits in budget.
    //     We scan ALL candidates rather than stopping at the first overflow, because a
    //     cheaper item appearing later may still fit within the remaining budget.
    const recommendations = computed(() => {
      let running = 0
      const result = []
      for (const c of candidates.value) {
        if (running + c.subtotal <= budget.value) {
          running += c.subtotal
          result.push(c)
        }
      }
      return result
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.subtotal, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    // --- Data loading ---
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData
        inventory.value = inventoryData
      } catch (err) {
        error.value = t('common.error') + ': ' + err.message
        console.error('Restocking load error:', err)
      } finally {
        loading.value = false
      }
    }

    // --- Place order ---
    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      submitting.value = true
      error.value = null
      try {
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.item_sku,
            name: r.item_name,
            quantity: r.restock_quantity,
            unit_price: r.unit_cost
          })),
          total_value: totalCost.value
        }
        const response = await api.createRestockingOrder(payload)
        submittedOrderNumber.value = response.order_number

        // Auto-clear the success banner after 8 seconds
        setTimeout(() => {
          submittedOrderNumber.value = null
        }, 8000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error('Restocking order error:', err)
      } finally {
        submitting.value = false
      }
    }

    // --- Trend badge: mirror Demand.vue color semantics ---
    // increasing -> success (green), stable -> info (blue), decreasing -> warning (amber)
    const getTrendBadgeClass = (trend) => {
      const map = { increasing: 'success', stable: 'info', decreasing: 'warning' }
      return map[trend] ?? 'info'
    }

    onMounted(loadData)

    return {
      t,
      currentCurrency,
      budget,
      loading,
      error,
      submitting,
      submittedOrderNumber,
      recommendations,
      totalCost,
      budgetRemaining,
      placeOrder,
      getTrendBadgeClass,
      formatCurrency
    }
  }
}
</script>

<style scoped>
/* Budget card body: slider on the left (~60%), stat blocks on the right */
.budget-body {
  display: grid;
  grid-template-columns: 3fr 2fr;
  gap: 2rem;
  align-items: center;
}

.budget-slider-block {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-slider-label {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

/* Label style matched to FilterBar .filter-group label */
.budget-slider-block label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
}

.budget-slider {
  width: 100%;
  accent-color: #0f172a;
  cursor: pointer;
  height: 6px;
}

.slider-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-stats {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-stat-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.875rem 1.125rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.25rem;
}

.budget-stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-stat-card.over-budget .budget-stat-value {
  color: #dc2626;
}

/* Card description (sub-title under card-title in the recommendations header) */
.card-description {
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.25rem;
}

/* Empty-state row */
.empty-state {
  color: #64748b;
  text-align: center;
  padding: 2rem;
  font-style: italic;
}

/* Place Order button footer */
.place-order-footer {
  display: flex;
  justify-content: flex-end;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.btn-primary {
  background: #0f172a;
  color: white;
  padding: 0.6rem 1.5rem;
  border-radius: 6px;
  border: none;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1e293b;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}
</style>
