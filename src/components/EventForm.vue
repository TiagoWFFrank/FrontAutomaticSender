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
          <label class="event-form__label">Período de evento</label>
          <button type="button" class="btn btn--secondary" @click="showPeriodModal = true">
            Definir período
          </button>
        </div>

        <div
          v-for="day in sortedSelectedDays"
          :key="day"
          class="event-form__group"
        >
          <label :for="`template-${day}`" class="event-form__label">
            Template da mensagem de {{ dayLabel(day) }}
          </label>
          <textarea
            :id="`template-${day}`"
            v-model="eventForm.messageTemplates[day]"
            class="event-form__textarea"
            :placeholder="`Mensagem para ${dayLabel(day)}`"
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

        <div class="event-form__group">
          <label for="attachment" class="event-form__label">Anexo</label>
          <input
            id="attachment"
            type="file"
            class="event-form__input"
            @change="handleFileChange"
            required
          />
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
  name: 'EventForm',
  
  data() {
    const today = new Date().toISOString().split('T')[0]
    return {
      // Form data
      eventForm: {
        eventId: '',
        daysOfWeek: [],
        time: '09:00',
        startDate: today,
        endDate: today,
        messageTemplates: {},
        enabled: true,
        attachment: null
      },
      
      // UI State
      isSubmitting: false,
      submitStatus: {
        show: false,
        type: '', // 'success' | 'error'
        message: ''
      },
      errors: {},

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
    sortedSelectedDays() {
      return [...this.eventForm.daysOfWeek].sort(
        (a, b) =>
          this.daysOptions.findIndex(d => d.value === a) -
          this.daysOptions.findIndex(d => d.value === b)
      )
    },
    isFormValid() {
      return (
        this.eventForm.eventId &&
        this.eventForm.daysOfWeek.length > 0 &&
        this.eventForm.time &&
        this.eventForm.startDate &&
        this.eventForm.endDate &&
        this.eventForm.attachment &&
        this.eventForm.daysOfWeek.every(
          day =>
            this.eventForm.messageTemplates[day] &&
            this.eventForm.messageTemplates[day].trim()
        )
      )
    }
  },

  watch: {
    'eventForm.daysOfWeek'(newDays) {
      for (const day of Object.keys(this.eventForm.messageTemplates)) {
        if (!newDays.includes(day)) {
          delete this.eventForm.messageTemplates[day]
        }
      }
    }
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

    // Validação
    validateForm() {
      this.errors = {}

      if (!this.eventForm.eventId.trim()) {
        this.errors.eventId = 'ID do evento é obrigatório'
      }

      return Object.keys(this.errors).length === 0
    },

    handleFileChange(event) {
      const file = event.target.files[0]
      if (file) {
        const reader = new FileReader()
        reader.onload = () => {
          this.eventForm.attachment = reader.result
        }
        reader.readAsDataURL(file)
      } else {
        this.eventForm.attachment = null
      }
    },

    // Submissão do formulário
    async handleSubmit() {
      if (!this.validateForm()) {
        return
      }

      this.isSubmitting = true
      this.hideAlert()

      try {
        const cleanedData = { ...this.eventForm }

        const response = await this.submitEventToAWS(cleanedData)

        if (response.status === 200) {
          this.showAlert('success', 'Evento cadastrado com sucesso!')
          this.resetForm()
          this.$emit('event-saved')
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

    // Utilitários
    resetForm() {
      const today = new Date().toISOString().split('T')[0]
      this.eventForm = {
        eventId: '',
        daysOfWeek: [],
        time: '09:00',
        startDate: today,
        endDate: today,
        messageTemplates: {},
        enabled: true,
        attachment: null
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

    dayLabel(value) {
      const match = this.daysOptions.find(d => d.value === value)
      return match ? match.label : value
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
  padding: 0;
  color: #111;
}

@media (max-width: 600px) {
  .event-registration {
    padding: var(--space-medium);
  }

  .event-form {
    padding: var(--space-medium);
  }

  .modal__content {
    width: 95%;
    max-width: none;
    padding: var(--space-medium);
  }

  .modal__actions {
    flex-direction: column;
    gap: var(--space-small);
  }

  .modal__actions .btn {
    width: 100%;
  }

  .event-form__actions .btn {
    width: 100%;
  }
}

@media (min-width: 1200px) {
  .event-registration {
    max-width: 1200px;
  }

  .event-form {
    max-width: 1200px;
  }
}
</style>
