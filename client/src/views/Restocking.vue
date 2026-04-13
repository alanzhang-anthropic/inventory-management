<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-readout">{{ formatCurrency(budget) }}</div>
          <input
            type="range"
            class="budget-slider"
            :min="minBudget"
            :max="maxBudget"
            :step="stepBudget"
            v-model.number="budget"
          />
          <div class="budget-help">{{ t('restocking.budgetHelp') }}</div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>

      <!-- Submit Error -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }}</h3>
        </div>

        <div v-if="candidates.length === 0" class="empty-state">
          {{ t('restocking.noCandidates') }}
        </div>
        <div v-else-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.budgetTooLow') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.recommendedQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.currentDemand }}</td>
                <td>{{ item.forecastedDemand }}</td>
                <td>{{ item.quantity }}</td>
                <td>{{ formatCurrency(item.unitCost) }}</td>
                <td><strong>{{ formatCurrency(item.lineTotal) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary + Place Order -->
        <div class="order-footer">
          <div class="summary-grid">
            <div class="stat-card">
              <div class="stat-label">{{ t('restocking.summary.items') }}</div>
              <div class="stat-value">{{ lineCount }}</div>
            </div>
            <div class="stat-card">
              <div class="stat-label">{{ t('restocking.summary.totalCost') }}</div>
              <div class="stat-value">{{ formatCurrency(totalCost) }}</div>
            </div>
            <div class="stat-card">
              <div class="stat-label">{{ t('restocking.summary.remaining') }}</div>
              <div class="stat-value">{{ formatCurrency(remainingBudget) }}</div>
            </div>
          </div>
          <button
            class="place-order-btn"
            :class="{ disabled: recommendations.length === 0 || isSubmitting }"
            :disabled="recommendations.length === 0 || isSubmitting"
            @click="placeOrder"
          >
            {{ isSubmitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(0)
    const minBudget = 0
    const stepBudget = 1000

    const isSubmitting = ref(false)
    const successMessage = ref('')
    const submitError = ref('')
    let successTimer = null

    const currencySymbol = computed(() =>
      currentCurrency.value === 'JPY' ? '\u00a5' : '$'
    )

    const formatCurrency = (value) =>
      `${currencySymbol.value}${value.toLocaleString()}`

    // Build candidates: increasing-trend forecasts matched to inventory
    const candidates = computed(() => {
      const inventoryMap = new Map(
        inventoryItems.value.map((item) => [item.sku, item])
      )

      return allForecasts.value
        .filter((f) => f.trend === 'increasing')
        .reduce((acc, f) => {
          const inv = inventoryMap.get(f.item_sku)
          if (!inv) return acc
          const gap = f.forecasted_demand - f.current_demand
          if (gap <= 0) return acc
          const unitCost = inv.unit_cost
          acc.push({
            sku: f.item_sku,
            name: f.item_name,
            currentDemand: f.current_demand,
            forecastedDemand: f.forecasted_demand,
            gap,
            unitCost,
            gapCost: gap * unitCost
          })
          return acc
        }, [])
        .sort((a, b) => b.gap - a.gap)
    })

    const totalMaxSpend = computed(() =>
      candidates.value.reduce((sum, c) => sum + c.gapCost, 0)
    )

    const maxBudget = computed(() =>
      Math.max(10000, totalMaxSpend.value)
    )

    // Set initial budget once candidates become available
    watch(candidates, (newCandidates) => {
      if (newCandidates.length > 0 && budget.value === 0) {
        budget.value = Math.round(totalMaxSpend.value / 2)
      }
    })

    const recommendations = computed(() => {
      let runningTotal = 0
      const picked = []
      for (const c of candidates.value) {
        if (runningTotal + c.gapCost <= budget.value) {
          picked.push({ ...c, quantity: c.gap, lineTotal: c.gapCost })
          runningTotal += c.gapCost
        }
      }
      return picked
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.lineTotal, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const lineCount = computed(() => recommendations.value.length)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || isSubmitting.value) return
      isSubmitting.value = true
      submitError.value = ''
      successMessage.value = ''
      if (successTimer) clearTimeout(successTimer)

      const filters = getCurrentFilters()
      const payload = {
        items: recommendations.value.map((r) => ({
          sku: r.sku,
          name: r.name,
          quantity: r.quantity,
          unit_price: r.unitCost
        })),
        warehouse: filters.warehouse !== 'all' ? filters.warehouse : undefined,
        category: filters.category !== 'all' ? filters.category : undefined
      }

      try {
        const resp = await api.submitRestockingOrder(payload)
        successMessage.value = t('restocking.orderSubmitted', {
          orderNumber: resp.order_number,
          days: 14
        })
        successTimer = setTimeout(() => {
          successMessage.value = ''
        }, 5000)
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + err.message
        console.error(err)
      } finally {
        isSubmitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      currencySymbol,
      loading,
      error,
      budget,
      minBudget,
      maxBudget,
      stepBudget,
      recommendations,
      totalCost,
      remainingBudget,
      lineCount,
      candidates,
      isSubmitting,
      successMessage,
      submitError,
      placeOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-controls {
  padding: 1rem 1.5rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-readout {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
  transition: background 0.15s, box-shadow 0.15s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
  box-shadow: 0 2px 8px rgba(59, 130, 246, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
}

.budget-help {
  font-size: 0.813rem;
  color: #64748b;
}

.empty-state {
  padding: 2rem 1.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

.order-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}

.place-order-btn {
  align-self: flex-start;
  padding: 0.625rem 1.5rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.place-order-btn:hover:not(.disabled) {
  background: #2563eb;
}

.place-order-btn.disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  margin-bottom: 1rem;
  padding: 0.875rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 6px;
  color: #065f46;
  font-weight: 500;
  font-size: 0.938rem;
}
</style>
