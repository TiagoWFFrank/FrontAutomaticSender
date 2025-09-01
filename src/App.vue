<template>
  <div class="event-registration">
    <div class="event-registration__header">
      <h1 class="event-registration__title">Cadastro de Eventos</h1>
      <p class="event-registration__description">
        Configure eventos automatizados para envio via WhatsApp
      </p>
    </div>

    <div class="event-registration__content">
      <!-- Formulário de Cadastro -->
      <form @submit.prevent="handleSubmit" class="event-form">
        <div class="event-form__group">
          <label for="eventId" class="event-form__label">ID do Evento</label>
          <select
            id="eventId"
            v-model="eventForm.eventId"
            @change="handleEventIdChange"
            class="event-form__input"
            :class="{ 'event-form__input--error': errors.eventId }"
            required
          >
            <option disabled value="">Selecione um evento</option>
            <option
              v-for="option in eventIdOptions"
              :key="option.value"
              :value="option.value"
            >
              {{ option.label }}
            </option>
            <option value="__new">Inserir um novo evento</option>
          </select>
          <span v-if="errors.eventId" class="event-form__error">
            {{ errors.eventId }}
          </span>
        </div>

        <div class="event-form__group">
          <label for="phoneNumbers" class="event-form__label">Números de Telefone</label>
          <div class="phone-input-container">
            <div
              v-for="(phone, index) in eventForm.phoneNumbers"
              :key="index"
              class="phone-input-row"
            >
              <input
                v-model="eventForm.phoneNumbers[index]"
                type="tel"
                class="event-form__input event-form__input--phone"
                placeholder="+5532999999999"
                required
              />
              <button
                type="button"
                @click="removePhoneNumber(index)"
                class="btn btn--danger btn--small"
                :disabled="eventForm.phoneNumbers.length === 1"
              >
                Remover
              </button>
            </div>
            <button
              type="button"
              @click="addPhoneNumber"
              class="btn btn--secondary"
            >
              + Adicionar Número
            </button>
          </div>
        </div>

        <div class="event-form__group">
          <label class="event-form__label">Período de evento</label>
          <button type="button" class="btn btn--secondary" @click="showPeriodModal = true">
            Definir período
          </button>
        </div>

        <div class="event-form__group">
          <label for="messageTemplate" class="event-form__label">Template da Mensagem</label>
          <textarea
            id="messageTemplate"
            v-model="eventForm.messageTemplate"
            class="event-form__textarea"
            placeholder="Bem-vindo! Qualquer dúvida, responda este WhatsApp."
            rows="4"
            required
          ></textarea>
        </div>

        <div class="event-form__group">
          <label class="event-form__checkbox-container">
            <input
              v-model="eventForm.enabled"
              type="checkbox"
              class="event-form__checkbox"
            />
            <span class="event-form__checkbox-label">Evento Ativo</span>
          </label>
        </div>

        <div class="event-form__actions">
          <button
            type="submit"
            class="btn btn--primary"
            :disabled="isSubmitting || !isFormValid"
            :class="{ 'btn--loading': isSubmitting }"
          >
            {{ isSubmitting ? 'Salvando...' : 'Salvar Evento' }}
          </button>
        </div>
      </form>

      <!-- Feedback Visual -->
      <div
        v-if="submitStatus.show"
        class="alert"
        :class="submitStatus.type === 'success' ? 'alert--success' : 'alert--error'"
      >
        <div class="alert__content">
          <h3 class="alert__title">
            {{ submitStatus.type === 'success' ? '✓ Sucesso!' : '✗ Erro!' }}
          </h3>
          <p class="alert__message">{{ submitStatus.message }}</p>
        </div>
      </div>

      <!-- Master Detail de Eventos -->
      <div v-if="existingEvents.length > 0" class="events-list">
        <h2 class="events-list__title">Eventos Cadastrados</h2>
        <div class="event-master-detail">
          <ul class="event-master">
            <li
              v-for="event in existingEvents"
              :key="event.eventId"
              @click="selectEvent(event)"
              :class="['event-master__item', { 'event-master__item--active': selectedEvent && selectedEvent.eventId === event.eventId }]"
            >
              {{ event.eventId }}
            </li>
          </ul>
          <div class="event-detail" v-if="selectedEvent">
            <div class="event-card">
              <div class="event-card__header">
                <h3 class="event-card__title">{{ selectedEvent.eventId }}</h3>
                <span
                  class="event-card__status"
                  :class="selectedEvent.enabled ? 'event-card__status--active' : 'event-card__status--inactive'"
                >
                  {{ selectedEvent.enabled ? 'Ativo' : 'Inativo' }}
                </span>
              </div>
              <p class="event-card__message">{{ selectedEvent.messageTemplate }}</p>
              <div class="event-card__details">
                <span class="event-card__time">🕒 {{ selectedEvent.time }}</span>
                <span class="event-card__days">📅 {{ formatDays(selectedEvent.daysOfWeek) }}</span>
                <span class="event-card__phones">📱 {{ selectedEvent.phoneNumbers.join(', ') }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
      </div>

    <!-- Modal Novo Evento -->
  <div v-if="showNewEventModal" class="modal">
    <div class="modal__content">
      <h3 class="modal__title">Inserir novo evento</h3>
      <input v-model="newEventId" type="text" class="event-form__input" placeholder="ID do evento" />
      <div class="modal__actions">
        <button type="button" class="btn btn--secondary" @click="closeNewEventModal">Cancelar</button>
        <button type="button" class="btn btn--primary" @click="confirmNewEvent">Adicionar</button>
      </div>
    </div>
  </div>

  <!-- Modal Período de Evento -->
  <div v-if="showPeriodModal" class="modal">
    <div class="modal__content">
      <h3 class="modal__title">Período de evento</h3>
      <div class="event-form__group">
        <label class="event-form__label">Dias da Semana</label>
        <div class="days-selector">
          <label v-for="day in daysOptions" :key="day.value" class="days-selector__item">
            <input
              v-model="eventForm.daysOfWeek"
              type="checkbox"
              :value="day.value"
              class="days-selector__checkbox"
            />
            <span class="days-selector__label">{{ day.label }}</span>
          </label>
        </div>
      </div>
      <div class="event-form__group">
        <label for="time" class="event-form__label">Horário</label>
        <input id="time" v-model="eventForm.time" type="time" class="event-form__input" />
      </div>
      <div class="event-form__group">
        <label for="startDate" class="event-form__label">Início</label>
        <input id="startDate" v-model="eventForm.startDate" type="date" class="event-form__input" />
      </div>
      <div class="event-form__group">
        <label for="endDate" class="event-form__label">Fim</label>
        <input id="endDate" v-model="eventForm.endDate" type="date" class="event-form__input" />
      </div>
      <div class="modal__actions">
        <button type="button" class="btn btn--secondary" @click="closePeriodModal">Cancelar</button>
        <button type="button" class="btn btn--primary" @click="closePeriodModal">Salvar</button>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'EventRegistration',
  
  data() {
    const today = new Date().toISOString().split('T')[0]
    return {
      // Form data
      eventForm: {
        eventId: '',
        phoneNumbers: [''],
        daysOfWeek: [],
        time: '09:00',
        startDate: today,
        endDate: today,
        messageTemplate: '',
        enabled: true
      },
      
      // UI State
      isSubmitting: false,
      submitStatus: {
        show: false,
        type: '', // 'success' | 'error'
        message: ''
      },
      errors: {},
      existingEvents: [],
      selectedEvent: null,

      eventIdOptions: [
        { value: 'boas_vindas', label: 'Boas Vindas' },
        { value: 'aniversario', label: 'Aniversário' },
        { value: 'promocao_semana', label: 'Promoção da Semana' }
      ],
      showNewEventModal: false,
      newEventId: '',
      showPeriodModal: false,

      // Options
      daysOptions: [
        { value: 'MON', label: 'Segunda' },
        { value: 'TUE', label: 'Terça' },
        { value: 'WED', label: 'Quarta' },
        { value: 'THU', label: 'Quinta' },
        { value: 'FRI', label: 'Sexta' },
        { value: 'SAT', label: 'Sábado' },
        { value: 'SUN', label: 'Domingo' }
      ],
      
      // TODO: Será fornecido pelo usuário
      API_CONFIG: {
        baseURL: 'https://sua-api-gateway.amazonaws.com/prod', // SUBSTITUIR
        authToken: 'seu-token-aqui' // SUBSTITUIR
      }
    }
  },

  computed: {
    isFormValid() {
      return (
        this.eventForm.eventId &&
        this.eventForm.phoneNumbers.some(phone => phone.trim()) &&
        this.eventForm.daysOfWeek.length > 0 &&
        this.eventForm.time &&
        this.eventForm.startDate &&
        this.eventForm.endDate &&
        this.eventForm.messageTemplate
      )
    }
  },

  mounted() {
    this.loadExistingEvents()
  },

  methods: {
    // Gerenciamento do ID do evento
    handleEventIdChange() {
      if (this.eventForm.eventId === '__new') {
        this.eventForm.eventId = ''
        this.newEventId = ''
        this.showNewEventModal = true
      }
    },
    closeNewEventModal() {
      this.showNewEventModal = false
      this.eventForm.eventId = ''
    },
    confirmNewEvent() {
      const trimmed = this.newEventId.trim()
      if (trimmed) {
        this.eventIdOptions.push({ value: trimmed, label: trimmed })
        this.eventForm.eventId = trimmed
        this.showNewEventModal = false
      }
    },

    closePeriodModal() {
      this.showPeriodModal = false
    },

    // Gerenciamento de telefones
    addPhoneNumber() {
      this.eventForm.phoneNumbers.push('')
    },
    
    removePhoneNumber(index) {
      if (this.eventForm.phoneNumbers.length > 1) {
        this.eventForm.phoneNumbers.splice(index, 1)
      }
    },

    // Validação
    validateForm() {
      this.errors = {}
      
      if (!this.eventForm.eventId.trim()) {
        this.errors.eventId = 'ID do evento é obrigatório'
      }
      
      // Validar números de telefone
      const validPhones = this.eventForm.phoneNumbers.filter(phone => {
        return phone.trim() && phone.match(/^\+\d{13}$/)
      })
      
      if (validPhones.length === 0) {
        this.errors.phoneNumbers = 'Pelo menos um número válido é obrigatório (+5532999999999)'
      }
      
      return Object.keys(this.errors).length === 0
    },

    // Submissão do formulário
    async handleSubmit() {
      if (!this.validateForm()) {
        return
      }

      this.isSubmitting = true
      this.hideAlert()

      try {
        // Limpar números vazios
        const cleanedData = {
          ...this.eventForm,
          phoneNumbers: this.eventForm.phoneNumbers.filter(phone => phone.trim())
        }

        const response = await this.submitEventToAWS(cleanedData)
        
        if (response.status === 200) {
          this.showAlert('success', 'Evento cadastrado com sucesso!')
          this.resetForm()
          await this.loadExistingEvents()
        }
        
      } catch (error) {
        console.error('Erro ao cadastrar evento:', error)
        this.showAlert('error', 'Erro ao cadastrar evento. Tente novamente.')
      } finally {
        this.isSubmitting = false
      }
    },

    // Requisição para AWS Lambda
    async submitEventToAWS(eventData) {
      const config = {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.API_CONFIG.authToken}` // ou formato necessário
        }
      }

      return await axios.post(
        `${this.API_CONFIG.baseURL}/events`, // ou endpoint específico
        eventData,
        config
      )
    },

    // Carregar eventos existentes (opcional)
    async loadExistingEvents() {
      try {
        const config = {
          headers: {
            'Authorization': `Bearer ${this.API_CONFIG.authToken}`
          }
        }

        const response = await axios.get(
          `${this.API_CONFIG.baseURL}/events`,
          config
        )

        if (response.status === 200 && response.data) {
          this.existingEvents = response.data
          this.selectedEvent = this.existingEvents[0] || null
        }
      } catch (error) {
        console.error('Erro ao carregar eventos:', error)
      }
    },

    selectEvent(event) {
      this.selectedEvent = event
    },

    // Utilitários
    resetForm() {
      const today = new Date().toISOString().split('T')[0]
      this.eventForm = {
        eventId: '',
        phoneNumbers: [''],
        daysOfWeek: [],
        time: '09:00',
        startDate: today,
        endDate: today,
        messageTemplate: '',
        enabled: true
      }
      this.errors = {}
    },

    showAlert(type, message) {
      this.submitStatus = {
        show: true,
        type,
        message
      }
      
      // Auto-hide após 5 segundos
      setTimeout(() => {
        this.hideAlert()
      }, 5000)
    },

    hideAlert() {
      this.submitStatus.show = false
    },

    formatDays(days) {
      const dayMap = {
        MON: 'Seg', TUE: 'Ter', WED: 'Qua',
        THU: 'Qui', FRI: 'Sex', SAT: 'Sáb', SUN: 'Dom'
      }
      return days.map(day => dayMap[day]).join(', ')
    }
  }
}
</script>

<style>
.event-registration {
  padding: var(--space-large);
  max-width: 800px;
  margin: 0 auto;
}

.event-registration__header {
  margin-bottom: var(--space-large);
  text-align: center;
}

.event-registration__title {
  font-size: var(--font-size-xxlarge);
  font-weight: var(--font-weight-bold);
  margin-bottom: var(--space-small);
  color: var(--s-12);
}

.event-registration__description {
  color: var(--s-11);
  font-size: var(--font-size-large);
}

/* Form Styles */
.event-form {
  background: var(--s-2);
  padding: var(--space-large);
  border-radius: var(--border-radius-large);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-large);
}

.event-form__group {
  margin-bottom: var(--space-medium);
}

.event-form__label {
  display: block;
  margin-bottom: var(--space-small);
  font-weight: var(--font-weight-medium);
  color: var(--s-12);
}

.event-form__input,
.event-form__textarea {
  background-color: var(--s-3);
  color: var(--s-12);
  width: 100%;
  padding: var(--space-one);
  border: 1px solid var(--s-5);
  border-radius: var(--border-radius-medium);
  font-size: var(--font-size-default);
}

.event-form__input:focus,
.event-form__textarea:focus {
  outline: none;
  border-color: var(--w-500);
  box-shadow: 0 0 0 1px var(--w-500);
}

.event-form__input--error {
  border-color: var(--r-500);
}

.event-form__error {
  color: var(--r-500);
  font-size: var(--font-size-small);
  margin-top: var(--space-small);
}

/* Phone Numbers */
.phone-input-row {
  display: flex;
  gap: var(--space-small);
  margin-bottom: var(--space-small);
}

.event-form__input--phone {
  flex: 1;
}

/* Days Selector */
.days-selector {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-medium);
}

.days-selector__item {
  display: flex;
  align-items: center;
  gap: var(--space-small);
  cursor: pointer;
  color: var(--s-11);
}

/* Buttons */
.btn {
  padding: var(--space-one) var(--space-two);
  border: none;
  border-radius: var(--border-radius-medium);
  font-weight: var(--font-weight-medium);
  cursor: pointer;
  transition: all 0.2s;
}

.btn--primary {
  background-color: var(--w-500);
  color: white;
}

.btn--primary:hover:not(:disabled) {
  background-color: var(--w-600);
}

.btn--secondary {
  background-color: var(--s-4);
  border: 1px solid var(--s-6);
  color: var(--s-12);
}

.btn--danger {
  background-color: var(--r-500);
  color: white;
}

.btn--small {
  padding: var(--space-smaller) var(--space-one);
  font-size: var(--font-size-small);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn--loading {
  opacity: 0.8;
}

/* Alerts */
.alert {
  padding: var(--space-medium);
  border-radius: var(--border-radius-medium);
  margin-bottom: var(--space-large);
}

.alert--success {
  background-color: var(--g-50);
  border: 1px solid var(--g-200);
  color: var(--g-900);
}

.alert--error {
  background-color: var(--r-50);
  border: 1px solid var(--r-200);
  color: var(--r-900);
}

.alert__title {
  font-weight: var(--font-weight-bold);
  margin-bottom: var(--space-small);
}

/* Events List */
.events-list {
  margin-top: var(--space-large);
}

.events-list__title {
  font-size: var(--font-size-xlarge);
  margin-bottom: var(--space-medium);
  color: var(--s-12);
}

.event-card {
  background: var(--s-2);
  padding: var(--space-medium);
  border-radius: var(--border-radius-large);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-medium);
}

.event-card__header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-small);
}

.event-card__title {
  font-weight: var(--font-weight-bold);
  color: var(--s-12);
}

.event-card__status {
  padding: var(--space-micro) var(--space-small);
  border-radius: var(--border-radius-large);
  font-size: var(--font-size-small);
  font-weight: var(--font-weight-medium);
}

.event-card__status--active {
  background-color: var(--g-100);
  color: var(--g-900);
}

.event-card__status--inactive {
  background-color: var(--r-100);
  color: var(--r-900);
}

.event-card__message {
  color: var(--s-11);
  margin-bottom: var(--space-small);
}

.event-card__details {
  display: flex;
  gap: var(--space-medium);
  font-size: var(--font-size-small);
  color: var(--s-10);
}

/* Master Detail */
.event-master-detail {
  display: flex;
  gap: var(--space-large);
}

.event-master {
  list-style: none;
  padding: 0;
  margin: 0;
  width: 200px;
}

.event-master__item {
  padding: var(--space-small);
  cursor: pointer;
  border-radius: var(--border-radius-medium);
  background: var(--s-3);
  margin-bottom: var(--space-small);
}

.event-master__item--active {
  background: var(--w-500);
  color: white;
}

.event-detail {
  flex: 1;
}

/* Modal */
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal__content {
  background: var(--s-2);
  padding: var(--space-large);
  border-radius: var(--border-radius-large);
  box-shadow: var(--shadow-sm);
  width: 100%;
  max-width: 400px;
}

.modal__title {
  margin-top: 0;
  margin-bottom: var(--space-medium);
  font-size: var(--font-size-large);
  color: var(--s-12);
}

.modal__actions {
  display: flex;
  justify-content: flex-end;
  gap: var(--space-small);
  margin-top: var(--space-medium);
}

/* Global variables and base styles */
:root {
  --space-large: 24px; --space-medium: 16px; --space-small: 8px; --space-one: 12px; --space-two: 16px; --space-smaller: 6px; --space-micro: 4px;
  --font-size-xxlarge: 28px; --font-size-xlarge: 22px; --font-size-large: 18px; --font-size-default: 16px; --font-size-small: 14px;
  --font-weight-bold: 700; --font-weight-medium: 600;
  --s-2: #ffffff; --s-3: #fafafa; --s-4: #f4f4f5; --s-5: #e4e4e7; --s-6: #d4d4d8; --s-10: #71717a; --s-11: #52525b; --s-12: #18181b;
  --w-500: #3b82f6; --w-600: #2563eb;
  --r-50: #fef2f2; --r-100: #fee2e2; --r-200: #fecaca; --r-500: #ef4444; --r-900: #7f1d1d;
  --g-50: #f0fdf4; --g-100: #dcfce7; --g-200: #bbf7d0; --g-900: #14532d;
  --border-radius-large: 14px; --border-radius-medium: 10px;
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.06);
}

body {
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Arial, "Apple Color Emoji", "Segoe UI Emoji";
  background: #f6f7f9;
  margin: 0;
  padding: 32px;
  color: #111;
}
</style>
