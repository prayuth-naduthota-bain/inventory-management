<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetSlider') }}</h3>
        </div>
        <div class="budget-content">
          <div class="budget-display">
            <div class="budget-amount">{{ formatCurrency(budget, selectedCurrency) }}</div>
            <input
              v-model.number="budget"
              type="range"
              min="0"
              max="50000"
              step="1000"
              class="budget-slider"
            />
            <div class="budget-range">
              <span>{{ formatCurrency(0, selectedCurrency) }}</span>
              <span>{{ formatCurrency(50000, selectedCurrency) }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Success Message -->
      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendations') }} ({{ recommendations.length }} {{ t('common.items') }})
          </h3>
          <div class="header-actions">
            <button
              v-if="recommendations.length > 0"
              @click="toggleSelectAll"
              class="action-button"
            >
              {{ allSelected ? t('restocking.deselectAll') : t('restocking.selectAll') }}
            </button>
          </div>
        </div>

        <div v-if="recommendations.length === 0" class="no-data">
          {{ t('restocking.noRecommendations') }}
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.select') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
                <th>{{ t('restocking.table.reason') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'selected-row': item.selected }"
              >
                <td>
                  <input
                    v-model="item.selected"
                    type="checkbox"
                    class="item-checkbox"
                  />
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ translateProductName(item.name) }}</td>
                <td><strong>{{ item.quantity }}</strong></td>
                <td>{{ formatCurrency(item.unitCost, selectedCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.totalCost, selectedCurrency) }}</strong></td>
                <td>
                  <span :class="['badge', item.reason === 'Backlog' ? 'danger' : 'info']">
                    {{ item.reason === 'Backlog' ? t('restocking.reasons.backlog') : t('restocking.reasons.highDemand') }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Section -->
        <div v-if="recommendations.length > 0" class="summary-section">
          <div class="summary-grid">
            <div class="summary-item">
              <span class="summary-label">{{ t('restocking.summary.selectedItems') }}:</span>
              <span class="summary-value">{{ selectedCount }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">{{ t('restocking.summary.totalCost') }}:</span>
              <span class="summary-value">{{ formatCurrency(selectedTotalCost, selectedCurrency) }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">{{ t('restocking.summary.remainingBudget') }}:</span>
              <span class="summary-value" :class="{ 'negative': remainingBudget < 0 }">
                {{ formatCurrency(remainingBudget, selectedCurrency) }}
              </span>
            </div>
          </div>

          <button
            @click="placeOrder"
            :disabled="selectedCount === 0 || submitting || remainingBudget < 0"
            class="place-order-button"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
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
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const { getCurrentFilters } = useFilters()

    // State
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    const budget = ref(25000) // Default budget: $25,000

    // Data from API
    const demandForecasts = ref([])
    const backlogItems = ref([])
    const inventoryItems = ref([])

    // Recommendations with selection state
    const recommendations = ref([])

    // Computed properties
    const selectedCount = computed(() => {
      return recommendations.value.filter(item => item.selected).length
    })

    const selectedTotalCost = computed(() => {
      return recommendations.value
        .filter(item => item.selected)
        .reduce((sum, item) => sum + item.totalCost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - selectedTotalCost.value
    })

    const allSelected = computed(() => {
      return recommendations.value.length > 0 &&
             recommendations.value.every(item => item.selected)
    })

    // Methods
    const loadData = async () => {
      try {
        loading.value = true
        error.value = null

        const filters = getCurrentFilters()

        // Fetch all necessary data in parallel
        const [demand, backlog, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getBacklog(),
          api.getInventory(filters)
        ])

        demandForecasts.value = demand
        backlogItems.value = backlog
        inventoryItems.value = inventory

        // Generate recommendations based on data
        generateRecommendations()
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error('Load error:', err)
      } finally {
        loading.value = false
      }
    }

    const generateRecommendations = () => {
      const filters = getCurrentFilters()
      const recs = []
      let remainingBudget = budget.value

      // Create a map of inventory items for quick lookup
      const inventoryMap = {}
      inventoryItems.value.forEach(item => {
        inventoryMap[item.sku] = item
      })

      // STEP 1: Prioritize backlog items first (items with existing customer orders)
      backlogItems.value.forEach(backlogItem => {
        const inventoryItem = inventoryMap[backlogItem.item_sku]

        // Skip if no inventory item found or doesn't match filters
        if (!inventoryItem) return

        // Apply warehouse filter
        if (filters.warehouse !== 'all' && inventoryItem.warehouse !== filters.warehouse) {
          return
        }

        // Apply category filter
        if (filters.category !== 'all' && inventoryItem.category !== filters.category) {
          return
        }

        const shortage = backlogItem.quantity_needed - backlogItem.quantity_available
        const quantity = Math.max(shortage, 0)

        if (quantity > 0) {
          const unitCost = inventoryItem.unit_cost
          const totalCost = quantity * unitCost

          // Only add if within budget
          if (totalCost <= remainingBudget) {
            recs.push({
              sku: backlogItem.item_sku,
              name: backlogItem.item_name,
              quantity: quantity,
              unitCost: unitCost,
              totalCost: totalCost,
              reason: 'Backlog',
              selected: true, // Pre-select backlog items
              priority: backlogItem.priority,
              category: inventoryItem.category,
              warehouse: inventoryItem.warehouse
            })
            remainingBudget -= totalCost
          }
        }
      })

      // STEP 2: Add high-demand forecast items to fill remaining budget
      // Filter and sort demand forecasts by highest forecasted demand
      const filteredForecasts = demandForecasts.value
        .filter(forecast => {
          const inventoryItem = inventoryMap[forecast.item_sku]
          if (!inventoryItem) return false

          // Apply warehouse filter
          if (filters.warehouse !== 'all' && inventoryItem.warehouse !== filters.warehouse) {
            return false
          }

          // Apply category filter
          if (filters.category !== 'all' && inventoryItem.category !== filters.category) {
            return false
          }

          // Only include items with positive demand change
          return forecast.forecasted_demand > forecast.current_demand
        })
        .sort((a, b) => b.forecasted_demand - a.forecasted_demand)

      filteredForecasts.forEach(forecast => {
        // Skip if already in backlog recommendations
        if (recs.some(r => r.sku === forecast.item_sku)) {
          return
        }

        const inventoryItem = inventoryMap[forecast.item_sku]
        if (!inventoryItem) return

        // Calculate quantity based on demand increase
        const demandIncrease = forecast.forecasted_demand - forecast.current_demand
        const quantity = Math.ceil(demandIncrease)

        if (quantity > 0) {
          const unitCost = inventoryItem.unit_cost
          const totalCost = quantity * unitCost

          // Only add if within remaining budget
          if (totalCost <= remainingBudget) {
            recs.push({
              sku: forecast.item_sku,
              name: forecast.item_name,
              quantity: quantity,
              unitCost: unitCost,
              totalCost: totalCost,
              reason: 'High Demand',
              selected: true, // Pre-select high demand items
              trend: forecast.trend,
              category: inventoryItem.category,
              warehouse: inventoryItem.warehouse
            })
            remainingBudget -= totalCost
          }
        }
      })

      recommendations.value = recs
    }

    const toggleSelectAll = () => {
      const newState = !allSelected.value
      recommendations.value.forEach(item => {
        item.selected = newState
      })
    }

    const placeOrder = async () => {
      if (selectedCount.value === 0) {
        return
      }

      if (remainingBudget.value < 0) {
        error.value = 'Total cost exceeds budget. Please adjust your selections.'
        return
      }

      try {
        submitting.value = true
        error.value = null

        const selectedItems = recommendations.value.filter(item => item.selected)

        // Generate random delivery time (7-14 days)
        const minDays = 7
        const maxDays = 14
        const deliveryDays = Math.floor(Math.random() * (maxDays - minDays + 1)) + minDays
        const expectedDelivery = new Date()
        expectedDelivery.setDate(expectedDelivery.getDate() + deliveryDays)

        const orderData = {
          items: selectedItems.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.quantity,
            unit_cost: item.unitCost
          })),
          total_cost: selectedTotalCost.value,
          status: 'Processing',
          expected_delivery: expectedDelivery.toISOString().split('T')[0],
          order_date: new Date().toISOString().split('T')[0]
        }

        await api.createRestockingOrder(orderData)

        successMessage.value = t('restocking.success') + ' ' +
                               t('restocking.deliveryEstimate', { min: minDays, max: maxDays })

        // Clear selections and regenerate recommendations
        setTimeout(() => {
          successMessage.value = null
          generateRecommendations()
        }, 5000)

      } catch (err) {
        error.value = t('restocking.error') + ': ' + err.message
        console.error('Place order error:', err)
      } finally {
        submitting.value = false
      }
    }

    // Watch budget changes and regenerate recommendations
    watch(budget, () => {
      generateRecommendations()
    })

    // Watch filter changes and reload data
    const { selectedLocation, selectedCategory } = useFilters()
    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    onMounted(() => {
      loadData()
    })

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      selectedCount,
      selectedTotalCost,
      remainingBudget,
      allSelected,
      toggleSelectAll,
      placeOrder,
      submitting,
      successMessage,
      selectedCurrency: currentCurrency,
      formatCurrency,
      translateProductName
    }
  }
}
</script>

<style scoped>
.page-header {
  margin-bottom: 2rem;
}

.page-header h2 {
  margin: 0 0 0.5rem 0;
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
}

.page-header p {
  margin: 0;
  color: #64748b;
  font-size: 0.938rem;
}

.loading,
.error {
  padding: 2rem;
  text-align: center;
  font-size: 1rem;
}

.error {
  color: #ef4444;
  background: #fee;
  border: 1px solid #fcc;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.budget-card {
  margin-bottom: 1.5rem;
}

.budget-content {
  padding: 2rem;
}

.budget-display {
  max-width: 600px;
  margin: 0 auto;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  text-align: center;
  margin-bottom: 1.5rem;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  transition: background 0.2s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.budget-slider::-moz-range-thumb {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  transition: background 0.2s ease;
}

.budget-slider::-moz-range-thumb:hover {
  background: #2563eb;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.budget-range {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 600;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.5rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  font-weight: 600;
  text-align: center;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 600;
  color: #0f172a;
}

.header-actions {
  display: flex;
  gap: 0.75rem;
}

.action-button {
  padding: 0.5rem 1rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  cursor: pointer;
  transition: all 0.2s ease;
}

.action-button:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.no-data {
  padding: 3rem 2rem;
  text-align: center;
  color: #94a3b8;
  font-size: 0.938rem;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

thead {
  background: #f8fafc;
  border-bottom: 2px solid #e2e8f0;
}

th {
  padding: 0.875rem 1rem;
  text-align: left;
  font-weight: 600;
  color: #475569;
  font-size: 0.813rem;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

tbody tr {
  border-bottom: 1px solid #f1f5f9;
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

tbody tr.selected-row {
  background: #eff6ff;
}

tbody tr.selected-row:hover {
  background: #dbeafe;
}

td {
  padding: 1rem;
  color: #0f172a;
}

.item-checkbox {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #3b82f6;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.danger {
  background: #fee2e2;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.summary-section {
  padding: 1.5rem;
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.summary-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.summary-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.summary-value.negative {
  color: #ef4444;
}

.place-order-button {
  width: 100%;
  padding: 1rem 2rem;
  background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 6px rgba(59, 130, 246, 0.3);
}

.place-order-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(59, 130, 246, 0.4);
}

.place-order-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}
</style>
