<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts and budget</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Post-order confirmation -->
      <div v-if="placedOrder" class="card confirmation-card">
        <h3 style="color: #059669; font-size: 1.25rem; font-weight: 700; margin-bottom: 1rem;">Order Placed</h3>
        <p class="confirmation-order-number">{{ placedOrder.order_number }}</p>
        <p style="color: #64748b; font-size: 0.875rem;">
          Expected delivery:
          <strong>{{ formatDeliveryDate(placedOrder.expected_delivery) }}</strong>
        </p>
        <button class="reset-btn" @click="resetOrder">Place Another Order</button>
      </div>

      <!-- Active ordering UI -->
      <div v-else>
        <!-- Budget slider card -->
        <div class="card">
          <div class="budget-slider">
            <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 0.75rem;">
              <label style="font-weight: 600; color: #0f172a; font-size: 0.875rem;">Budget</label>
              <span class="budget-amount">{{ formatCurrency(budget) }}</span>
            </div>
            <input
              type="range"
              min="0"
              max="500000"
              step="5000"
              v-model.number="budget"
              style="width: 100%; accent-color: #2563eb;"
            />
            <div class="slider-labels">
              <span>$0</span>
              <span>$500,000</span>
            </div>
          </div>
        </div>

        <!-- Recommended Items card -->
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Recommended Items</h3>
            <span class="summary-text">
              {{ budgetSummary.itemsCount }} items &middot;
              {{ formatCurrency(budgetSummary.budgetUsed) }} of {{ formatCurrency(budgetSummary.budgetTotal) }} budget used
            </span>
          </div>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>SKU</th>
                  <th>Item Name</th>
                  <th>Current Demand</th>
                  <th>Forecasted Demand</th>
                  <th>Change %</th>
                  <th>Restock Qty</th>
                  <th>Est. Cost</th>
                </tr>
              </thead>
              <tbody>
                <template v-if="recommendations.length > 0">
                  <tr
                    v-for="item in recommendations"
                    :key="item.item_sku"
                    :class="{ 'row-over-budget': !item.withinBudget }"
                  >
                    <td><strong>{{ item.item_sku }}</strong></td>
                    <td>{{ item.item_name }}</td>
                    <td>{{ item.current_demand }}</td>
                    <td><strong>{{ item.forecasted_demand }}</strong></td>
                    <td>{{ formatChangePct(item.change_pct) }}</td>
                    <td>{{ item.demand_gap }}</td>
                    <td>
                      {{ formatCurrency(item.restock_cost) }}
                      <span v-if="item.withinBudget" class="badge success" style="margin-left: 0.5rem;">In Budget</span>
                      <span v-else style="color: #94a3b8; font-size: 0.75rem; margin-left: 0.5rem;">Over Budget</span>
                    </td>
                  </tr>
                </template>
                <tr v-else>
                  <td colspan="7" style="text-align: center; color: #64748b; padding: 2rem;">
                    No items match the current budget or forecast data.
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Place Order button -->
        <button
          class="place-order-btn"
          :disabled="budgetSummary.itemsCount === 0 || placingOrder"
          @click="placeOrder"
        >
          {{ placingOrder ? 'Placing...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(100000)
    const forecasts = ref([])
    const inventory = ref([])
    const loading = ref(false)
    const error = ref(null)
    const placingOrder = ref(false)
    const placedOrder = ref(null)

    // Greedy budget allocation sorted by change_pct descending
    const recommendations = computed(() => {
      // Build sku → unit_cost lookup from inventory
      const costLookup = {}
      inventory.value.forEach(item => {
        costLookup[item.sku] = item.unit_cost
      })

      // Build candidates with demand gap and cost
      const candidates = []
      forecasts.value.forEach(forecast => {
        const demand_gap = forecast.forecasted_demand - forecast.current_demand
        if (demand_gap <= 0) return

        const unit_cost = costLookup[forecast.item_sku]
        if (unit_cost == null) return

        const restock_cost = demand_gap * unit_cost
        const change_pct = (forecast.forecasted_demand - forecast.current_demand) / forecast.current_demand * 100

        candidates.push({
          id: forecast.id,
          item_sku: forecast.item_sku,
          item_name: forecast.item_name,
          current_demand: forecast.current_demand,
          forecasted_demand: forecast.forecasted_demand,
          trend: forecast.trend,
          period: forecast.period,
          demand_gap,
          unit_cost,
          restock_cost,
          change_pct
        })
      })

      // Sort descending by change_pct
      candidates.sort((a, b) => b.change_pct - a.change_pct)

      // Greedy fill within budget
      let remaining = budget.value
      return candidates.map(item => {
        let withinBudget = false
        if (item.restock_cost <= remaining) {
          withinBudget = true
          remaining -= item.restock_cost
        }
        return { ...item, withinBudget }
      })
    })

    const budgetSummary = computed(() => {
      const withinBudgetItems = recommendations.value.filter(r => r.withinBudget)
      return {
        itemsCount: withinBudgetItems.length,
        budgetUsed: withinBudgetItems.reduce((sum, r) => sum + r.restock_cost, 0),
        budgetTotal: budget.value
      }
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastsData
        inventory.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placingOrder.value = true
      try {
        const withinBudgetItems = recommendations.value.filter(r => r.withinBudget)
        const result = await api.createRestockingOrder({
          items: withinBudgetItems.map(r => ({
            item_sku: r.item_sku,
            item_name: r.item_name,
            quantity: r.demand_gap,
            unit_cost: r.unit_cost
          }))
        })
        placedOrder.value = result
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const resetOrder = () => {
      placedOrder.value = null
    }

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatChangePct = (pct) => {
      return `+${pct.toFixed(1)}%`
    }

    const formatDeliveryDate = (dateStr) => {
      const date = new Date(dateStr)
      if (isNaN(date.getTime())) return dateStr
      return date.toLocaleDateString()
    }

    onMounted(() => loadData())

    return {
      budget,
      forecasts,
      inventory,
      loading,
      error,
      placingOrder,
      placedOrder,
      recommendations,
      budgetSummary,
      placeOrder,
      resetOrder,
      formatCurrency,
      formatChangePct,
      formatDeliveryDate
    }
  }
}
</script>

<style scoped>
.restocking {}

.budget-slider {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.budget-amount {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.row-over-budget td {
  opacity: 0.45;
}

.summary-text {
  color: #64748b;
  font-size: 0.875rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.confirmation-card {
  padding: 2rem;
  text-align: center;
}

.confirmation-order-number {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0.5rem 0;
}

.reset-btn {
  border: 1px solid #e2e8f0;
  background: white;
  color: #374151;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  cursor: pointer;
  margin-top: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.2s ease;
}

.reset-btn:hover {
  background: #f8fafc;
}
</style>
