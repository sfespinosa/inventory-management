<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your budget and review recommended items based on current inventory reorder levels</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-slider-row">
          <input
            type="range"
            v-model.number="budget"
            :min="0"
            :max="500000"
            :step="1000"
            class="budget-slider"
          />
          <span class="budget-display">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="budget-summary-row">
          <div class="budget-summary-item">
            <span class="budget-summary-label">Budget</span>
            <span class="budget-summary-value">{{ formatCurrency(budget) }}</span>
          </div>
          <div class="budget-summary-divider"></div>
          <div class="budget-summary-item">
            <span class="budget-summary-label">Allocated</span>
            <span class="budget-summary-value">{{ formatCurrency(allocatedBudget) }}</span>
          </div>
          <div class="budget-summary-divider"></div>
          <div class="budget-summary-item">
            <span class="budget-summary-label">Remaining</span>
            <span
              class="budget-summary-value"
              :class="{ 'remaining-positive': remainingBudget >= 0, 'remaining-negative': remainingBudget < 0 }"
            >{{ formatCurrency(remainingBudget) }}</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items ({{ recommendedItems.length }} items, {{ formatCurrency(allocatedBudget) }} total)
          </h3>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget slider.
        </div>
        <div v-else class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Category</th>
                <th>Trend</th>
                <th class="col-numeric">Qty to Order</th>
                <th class="col-numeric">Unit Cost</th>
                <th class="col-numeric">Total Cost</th>
                <th class="col-numeric">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>{{ item.item_name }}</td>
                <td><code class="sku-code">{{ item.item_sku }}</code></td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-numeric">{{ item.order_quantity.toLocaleString() }}</td>
                <td class="col-numeric">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-numeric"><strong>{{ formatCurrency(item.item_cost) }}</strong></td>
                <td class="col-numeric">{{ item.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Row -->
      <div class="place-order-row">
        <button
          class="btn-primary"
          :disabled="recommendedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          <span v-if="submitting">Submitting...</span>
          <span v-else>Place Order ({{ recommendedItems.length }} items)</span>
        </button>
      </div>

      <!-- Success Banner -->
      <div v-if="successMessage" class="banner banner-success">
        {{ successMessage }}
      </div>

      <!-- Error Banner -->
      <div v-if="submitError" class="banner banner-error">
        {{ submitError }}
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

const LEAD_TIMES = {
  'Circuit Boards': 14,
  'Sensors': 10,
  'Actuators': 12,
  'Controllers': 21,
  'Power Supplies': 7,
}

function getLeadTime(category) {
  if (!category) return 14
  const key = Object.keys(LEAD_TIMES).find(
    k => k.toLowerCase() === category.toLowerCase()
  )
  return key ? LEAD_TIMES[key] : 14
}

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const demandItems = ref([])
    const enrichedItems = ref([])

    const budget = ref(100000)
    const submitting = ref(false)
    const successMessage = ref(null)
    const submitError = ref(null)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Fetch demand forecasts and inventory in parallel
        const [demandData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])

        // Build demand lookup by SKU so we can show trend info where it exists
        const demandBySku = {}
        for (const d of demandData) {
          demandBySku[d.item_sku] = d
        }

        // Candidates: inventory items at or below their reorder point
        enrichedItems.value = inventoryData
          .filter(inv => inv.quantity_on_hand <= inv.reorder_point)
          .map(inv => {
            const demand = demandBySku[inv.sku]
            return {
              item_sku: inv.sku,
              item_name: inv.name,
              category: inv.category || '',
              unit_cost: inv.unit_cost,
              quantity_on_hand: inv.quantity_on_hand,
              reorder_point: inv.reorder_point,
              // Use demand trend if a matching forecast exists, otherwise stable
              trend: demand ? demand.trend : 'stable',
              lead_time_days: getLeadTime(inv.category),
            }
          })
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Computed: greedy recommendation within budget
    const recommendedItems = computed(() => {
      const candidates = enrichedItems.value.map(item => {
        // Order enough to bring stock back up to the reorder point
        const order_quantity = item.reorder_point - item.quantity_on_hand
        const item_cost = order_quantity * item.unit_cost
        return { ...item, order_quantity, item_cost }
      })

      // Sort: increasing trend first, then by shortfall size descending
      candidates.sort((a, b) => {
        if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
        if (a.trend !== 'increasing' && b.trend === 'increasing') return 1
        return b.order_quantity - a.order_quantity
      })

      // Greedy selection within budget
      let remaining = budget.value
      const selected = []
      for (const item of candidates) {
        if (item.item_cost <= remaining) {
          selected.push(item)
          remaining -= item.item_cost
        }
      }
      return selected
    })

    const allocatedBudget = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - allocatedBudget.value)

    const formatCurrency = (value) => {
      return '$' + Math.round(value).toLocaleString('en-US')
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      successMessage.value = null
      submitError.value = null
      try {
        const payload = {
          items: recommendedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            category: i.category,
            quantity: i.order_quantity,
            unit_cost: i.unit_cost,
            lead_time_days: i.lead_time_days,
          })),
          total_budget: budget.value,
        }
        const response = await api.createRestockingOrder(payload)
        successMessage.value = `Restocking order ${response.order_number} submitted successfully. View it in the Orders tab.`
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      submitting,
      successMessage,
      submitError,
      recommendedItems,
      allocatedBudget,
      remainingBudget,
      formatCurrency,
      placeOrder,
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits main-content padding from App.vue */
}

/* Budget slider row */
.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-bottom: 1rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-display {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 110px;
  text-align: right;
  letter-spacing: -0.025em;
}

/* Budget summary row */
.budget-summary-row {
  display: flex;
  align-items: center;
  gap: 0;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
}

.budget-summary-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0.875rem 1rem;
  gap: 0.25rem;
}

.budget-summary-divider {
  width: 1px;
  height: 40px;
  background: #e2e8f0;
}

.budget-summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-summary-value {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
}

.remaining-positive {
  color: #059669;
}

.remaining-negative {
  color: #dc2626;
}

/* Recommendations table */
.recommendations-table {
  width: 100%;
}

.col-numeric {
  text-align: right;
}

.sku-code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #334155;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Place order row */
.place-order-row {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Banners */
.banner {
  padding: 1rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1rem;
}

.banner-success {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
}

.banner-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}
</style>
