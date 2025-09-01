<template>
  <div>
    <button class="btn btn--primary open-form-button" @click="showForm = true">
      Novo Evento
    </button>

    <table v-if="events.length" class="events-table">
      <thead>
        <tr>
          <th>Título do Evento</th>
          <th>Período</th>
          <th>Próxima Execução</th>
          <th>Ativo</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="event in events" :key="event.eventId" @click="selectEvent(event)">
          <td>{{ event.eventId }}</td>
          <td>{{ formatPeriod(event) }}</td>
          <td>{{ event.nextRun || '-' }}</td>
          <td><input type="checkbox" v-model="event.enabled" disabled /></td>
          <td>
            <button type="button" class="btn btn--secondary btn--small" @click.stop="editEvent(event)">
              Editar
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <div v-if="selectedEvent" class="clients-detail">
      <h2>Clientes de {{ selectedEvent.eventId }}</h2>
      <table class="clients-table">
        <thead>
          <tr>
            <th>Nome</th>
            <th>Número</th>
            <th>Enviado</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="client in selectedEvent.clients || []" :key="client.phone">
            <td>{{ client.name }}</td>
            <td>{{ client.phone }}</td>
            <td>{{ client.sent ? 'Sim' : 'Não' }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-if="showForm" class="form-modal">
      <div class="form-modal__content">
        <button type="button" class="form-modal__close btn btn--secondary btn--small" @click="closeForm">Fechar</button>
        <EventForm @event-saved="handleEventSaved" />
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import EventForm from './components/EventForm.vue'

export default {
  name: 'App',
  components: { EventForm },
  data() {
    return {
      showForm: false,
      events: [],
      selectedEvent: null,
      API_CONFIG: {
        baseURL: 'https://sua-api-gateway.amazonaws.com/prod',
        authToken: 'seu-token-aqui'
      }
    }
  },
  mounted() {
    this.loadEvents()
  },
  methods: {
    closeForm() {
      this.showForm = false
    },
    async loadEvents() {
      try {
        const config = {
          headers: {
            Authorization: `Bearer ${this.API_CONFIG.authToken}`
          }
        }
        const response = await axios.get(`${this.API_CONFIG.baseURL}/events`, config)
        if (response.status === 200 && Array.isArray(response.data)) {
          this.events = response.data
        }
      } catch (err) {
        console.error('Erro ao carregar eventos:', err)
      }
    },
    selectEvent(event) {
      this.selectedEvent = event
    },
    editEvent(event) {
      this.selectedEvent = event
      this.showForm = true
    },
    handleEventSaved() {
      this.showForm = false
      this.loadEvents()
    },
    formatPeriod(event) {
      if (event.startDate && event.endDate) {
        return `${event.startDate} - ${event.endDate}`
      }
      return '-'
    }
  }
}
</script>

<style>
.open-form-button {
  position: fixed;
  top: var(--space-large);
  right: var(--space-large);
}

.events-table,
.clients-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: var(--space-large);
}

.events-table th,
.events-table td,
.clients-table th,
.clients-table td {
  border: 1px solid var(--s-5);
  padding: var(--space-small);
  text-align: left;
}

.events-table tbody tr:hover {
  background: var(--s-4);
}

.clients-detail {
  margin-top: var(--space-large);
}

.form-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 900;
}

.form-modal__content {
  background: var(--s-2);
  padding: var(--space-large);
  border-radius: var(--border-radius-large);
  box-shadow: var(--shadow-sm);
  width: 100%;
  max-width: 800px;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
}

.form-modal__close {
  position: absolute;
  top: var(--space-small);
  right: var(--space-small);
}
</style>

