<template>
  <div>
    <header class="top-menu">
      <button class="btn btn--primary" @click="showForm = true">
        Novo Evento
      </button>
    </header>

    <main class="main-content">
      <DxDataGrid
        v-if="events.length"
        :data-source="events"
        key-expr="eventId"
        :show-borders="true"
      >
      <DxColumn data-field="eventId" caption="Título do Evento" />
      <DxColumn caption="Período">
        <template #cellTemplate="{ data }">
          {{ formatPeriod(data.data) }}
        </template>
      </DxColumn>
      <DxColumn data-field="nextRun" caption="Próxima Execução" />
      <DxColumn data-field="enabled" caption="Ativo" data-type="boolean" />
      <DxColumn caption="">
        <template #cellTemplate="{ data }">
          <button
            type="button"
            class="btn btn--secondary btn--small"
            @click.stop="editEvent(data.data)"
          >
            Editar
          </button>
        </template>
      </DxColumn>
        <DxMasterDetail :enabled="true">
          <template #template="{ data }">
            <DxDataGrid
              :data-source="data.data.clients || []"
              :show-borders="true"
            >
              <DxColumn data-field="name" caption="Nome" />
              <DxColumn data-field="phone" caption="Número" />
              <DxColumn data-field="sent" caption="Enviado" data-type="boolean" />
            </DxDataGrid>
          </template>
        </DxMasterDetail>
      </DxDataGrid>
    </main>

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
import { DxDataGrid, DxColumn, DxMasterDetail } from 'devextreme-vue/data-grid'

export default {
  name: 'App',
  components: { EventForm, DxDataGrid, DxColumn, DxMasterDetail },
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
      if (!this.events.length) {
        this.events = this.getSampleEvents()
      }
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
    },
    getSampleEvents() {
      return [
        {
          eventId: 'boas_vindas',
          startDate: '2024-01-01',
          endDate: '2024-12-31',
          nextRun: '2024-05-25 09:00',
          enabled: true,
          clients: [
            { name: 'João', phone: '+5531999999999', sent: true },
            { name: 'Maria', phone: '+5532888888888', sent: false }
          ]
        },
        {
          eventId: 'aniversarios',
          startDate: '2024-05-01',
          endDate: '2024-07-31',
          nextRun: '2024-05-30 10:00',
          enabled: false,
          clients: [
            { name: 'Carlos', phone: '+5532777777777', sent: false }
          ]
        }
      ]
    }
  }
}
</script>

<style>

.top-menu {
  display: flex;
  justify-content: flex-end;
  padding: var(--space-large);
  background: var(--s-2);
  box-shadow: var(--shadow-sm);
}

.main-content {
  padding: var(--space-large);
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

