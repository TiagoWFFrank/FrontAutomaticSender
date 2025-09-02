<template>
  <div class="events-page">
    <!-- Estado vazio com CTA -->
    <div v-if="!loading && !events.length" class="empty-state">
      <h2>Sem eventos cadastrados</h2>
      <p>Crie seu primeiro evento para começar os disparos automatizados.</p>
      <button class="btn btn--primary" @click="openCreateForm">Novo Evento</button>
      <button class="btn btn--secondary" @click="loadEvents">Recarregar</button>
    </div>

    <!-- Grid principal -->
    <DxDataGrid
      v-else
      :data-source="events"
      key-expr="eventId"
      :show-borders="true"
      :row-alternation-enabled="true"
      :column-auto-width="true"
      :allow-column-resizing="true"
      column-resizing-mode="widget"
      :hover-state-enabled="true"
      no-data-text="Nenhum evento encontrado"
      class="events-grid"
    >
      <!-- Toolbar -->
      <DxToolbar>
        <DxItem location="before" template="titleTemplate" />
        <DxItem location="after" widget="dxButton" :options="toolbarButtons.reload" />
        <DxItem location="after" widget="dxButton" :options="toolbarButtons.create" />
        <DxItem location="after" template="legendTemplate" />
        <DxItem location="after" name="searchPanel" />
      </DxToolbar>

      <template #titleTemplate>
        <div class="grid-title">
          <h2>Eventos Automatizados</h2>
        </div>
      </template>

      <template #legendTemplate>
        <div class="legend">
          <span class="legend__dot legend__dot--active"></span> Ativo
          <span class="legend__dot legend__dot--inactive"></span> Inativo
        </div>
      </template>

      <!-- UX do grid -->
      <DxSearchPanel :visible="true" :width="280" placeholder="Buscar por título…" />
      <DxFilterRow :visible="true" />
      <DxHeaderFilter :visible="true" />
      <DxColumnChooser :enabled="true" mode="select" />
      <DxStateStoring :enabled="true" type="localStorage" storage-key="events-grid-v1" />
      <DxPaging :page-size="10" />
      <DxPager :show-page-size-selector="true" :allowed-page-sizes="[10, 20, 50]" :show-info="true" :show-navigation-buttons="true" />
      <DxLoadPanel :enabled="true" />

      <!-- Colunas -->
      <DxColumn
        data-field="eventId"
        caption="Título do Evento"
        :fixed="true"
        :allow-sorting="true"
      />
      <DxColumn caption="Período" :calculate-cell-value="calcPeriod" />
      <DxColumn caption="Próxima Execução" :cell-template="'nextRunTpl'" />
      <template #nextRunTpl="{ data }">
        <span>{{ formatDateTime(data.data?.nextRun) || '-' }}</span>
      </template>

      <DxColumn
        data-field="enabled"
        caption="Ativo"
        data-type="boolean"
        :cell-template="'enabledTpl'"
        :width="120"
      />
      <template #enabledTpl="{ data }">
        <button
          class="btn btn--pill"
          :class="data.data.enabled ? 'btn--success' : 'btn--muted'"
          @click.stop="toggleEnabled(data.data)"
          :disabled="togglingId === data.data.eventId"
          title="Ativar/Desativar"
        >
          {{ data.data.enabled ? 'Ativo' : 'Inativo' }}
        </button>
      </template>

      <DxColumn caption="Ações" :width="220" :cell-template="'actionsTpl'" />
      <template #actionsTpl="{ data }">
        <div class="row-actions">
          <button
            type="button"
            class="btn btn--secondary btn--small"
            @click.stop="editEvent(data.data)"
            title="Editar"
          >
            Editar
          </button>
          <button
            type="button"
            class="btn btn--danger btn--small"
            @click.stop="confirmDelete(data.data)"
            title="Excluir"
          >
            Excluir
          </button>
        </div>
      </template>

      <!-- Master Detail: clientes do evento -->
      <DxMasterDetail :enabled="true" :template="'detailTpl'" />
      <template #detailTpl="{ data }">
        <DxDataGrid
          :data-source="(data.data && data.data.clients) || []"
          :show-borders="true"
          :row-alternation-enabled="true"
          no-data-text="Nenhum cliente vinculado"
          class="clients-grid"
        >
          <DxSearchPanel :visible="true" :width="240" placeholder="Buscar cliente…" />
          <DxHeaderFilter :visible="true" />
          <DxPaging :page-size="5" />
          <DxPager :show-info="true" :show-navigation-buttons="true" />
          <DxColumn data-field="name" caption="Nome" />
          <DxColumn data-field="phone" caption="Número" />
          <DxColumn data-field="sent" caption="Enviado" data-type="boolean" />
          <DxColumn caption="Ações" :width="180" :cell-template="'clientActionsTpl'"/>
          <template #clientActionsTpl="{ data: row }">
            <div class="row-actions">
              <button class="btn btn--secondary btn--xsmall" @click.stop="previewMessage(data.data, row.data)">Pré-visualizar</button>
              <button class="btn btn--primary btn--xsmall" @click.stop="sendNow(data.data, row.data)">Enviar agora</button>
            </div>
          </template>
        </DxDataGrid>
      </template>
    </DxDataGrid>

    <!-- Modal do formulário -->
    <div v-if="showForm" class="form-modal" @click.self="closeForm">
      <div class="form-modal__content">
        <button type="button" class="form-modal__close btn btn--secondary btn--small" @click="closeForm">Fechar</button>
        <!-- Passa o evento selecionado no modo edição -->
        <EventForm :value="selectedEvent" @event-saved="handleEventSaved" />
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import EventForm from './components/EventForm.vue'
import {
  DxDataGrid,
  DxColumn,
  DxMasterDetail,
  DxSearchPanel,
  DxToolbar,
  DxItem,
  DxPaging,
  DxPager,
  DxColumnChooser,
  DxFilterRow,
  DxHeaderFilter,
  DxLoadPanel,
  DxStateStoring,
} from 'devextreme-vue/data-grid'

const tz = 'America/Sao_Paulo'
const fmtDate = (d) => {
  if (!d) return ''
  try {
    // aceita Date, ISO string ou 'YYYY-MM-DD HH:mm'
    const asISO = typeof d === 'string' && d.includes(' ') ? d.replace(' ', 'T') : d
    const date = new Date(asISO)
    if (isNaN(date.getTime())) return d
    return date.toLocaleDateString('pt-BR', { timeZone: tz })
  } catch { return d }
}
const fmtDateTime = (d) => {
  if (!d) return ''
  try {
    const asISO = typeof d === 'string' && d.includes(' ') ? d.replace(' ', 'T') : d
    const date = new Date(asISO)
    if (isNaN(date.getTime())) return d
    return date.toLocaleString('pt-BR', {
      timeZone: tz,
      year: 'numeric', month: '2-digit', day: '2-digit',
      hour: '2-digit', minute: '2-digit'
    })
  } catch { return d }
}

export default {
  name: 'App',
  components: {
    EventForm,
    DxDataGrid,
    DxColumn,
    DxMasterDetail,
    DxSearchPanel,
    DxToolbar,
    DxItem,
    DxPaging,
    DxPager,
    DxColumnChooser,
    DxFilterRow,
    DxHeaderFilter,
    DxLoadPanel,
    DxStateStoring,
  },
  data() {
    return {
      loading: false,
      showForm: false,
      events: [],
      selectedEvent: null,
      togglingId: null,
      deletingId: null,
      API_CONFIG: {
        baseURL: 'https://sua-api-gateway.amazonaws.com/prod',
        authToken: 'seu-token-aqui',
      },
      api: null,
      toolbarButtons: {
        create: {
          icon: 'add',
          text: 'Novo Evento',
          type: 'default',
          onClick: () => this.openCreateForm(),
        },
        reload: {
          icon: 'refresh',
          text: 'Recarregar',
          type: 'normal',
          onClick: () => this.loadEvents(),
        },
      },
    }
  },
  created() {
    // instancia dedicada do axios com auth
    this.api = axios.create({
      baseURL: this.API_CONFIG.baseURL,
      headers: { Authorization: `Bearer ${this.API_CONFIG.authToken}` },
    })
  },
  mounted() {
    this.loadEvents()
    window.addEventListener('keydown', this.onKeydown)
  },
  beforeUnmount() {
    window.removeEventListener('keydown', this.onKeydown)
  },
  methods: {
    onKeydown(e) {
      if (e.key === 'Escape' && this.showForm) this.closeForm()
    },
    openCreateForm() {
      this.selectedEvent = null
      this.showForm = true
    },
    closeForm() {
      this.showForm = false
      this.selectedEvent = null
    },
    async loadEvents() {
      this.loading = true
      try {
        const { status, data } = await this.api.get('/events')
        if (status === 200 && Array.isArray(data)) {
          // normaliza datas
          this.events = data.map(ev => ({
            ...ev,
            startDate: ev.startDate || null,
            endDate: ev.endDate || null,
            nextRun: ev.nextRun || null,
            enabled: !!ev.enabled,
            clients: Array.isArray(ev.clients) ? ev.clients : [],
          }))
        } else {
          this.events = []
        }
      } catch (err) {
        console.error('Erro ao carregar eventos:', err)
        // fallback demo (opcional)
        if (!this.events.length) this.events = this.getSampleEvents()
      } finally {
        this.loading = false
      }
    },
    editEvent(event) {
      this.selectedEvent = { ...event } // evita mutação direta
      this.showForm = true
    },
    async toggleEnabled(event) {
      if (!event?.eventId) return
      this.togglingId = event.eventId
      try {
        const updated = { ...event, enabled: !event.enabled }
        // Chamada exemplo: PUT /events/:id
        await this.api.put(`/events/${encodeURIComponent(event.eventId)}`, updated)
        // atualiza localmente
        const idx = this.events.findIndex(e => e.eventId === event.eventId)
        if (idx >= 0) this.$set(this.events, idx, updated)
      } catch (err) {
        console.error('Erro ao alternar ativo:', err)
        alert('Não foi possível atualizar o status do evento.')
      } finally {
        this.togglingId = null
      }
    },
    async confirmDelete(event) {
      if (!event?.eventId || this.deletingId) return
      const ok = confirm(`Excluir o evento "${event.eventId}"? Esta ação não pode ser desfeita.`)
      if (!ok) return
      this.deletingId = event.eventId
      try {
        await this.api.delete(`/events/${encodeURIComponent(event.eventId)}`)
        this.events = this.events.filter(e => e.eventId !== event.eventId)
      } catch (err) {
        console.error('Erro ao excluir:', err)
        alert('Não foi possível excluir o evento.')
      } finally {
        this.deletingId = null
      }
    },
    handleEventSaved(payload) {
      // payload opcional: evento salvo
      this.showForm = false
      this.selectedEvent = null
      // recarrega para refletir
      this.loadEvents()
    },
    // Formata "Período"
    calcPeriod(row) {
      if (!row) return '-'
      const s = row.startDate ? fmtDate(row.startDate) : null
      const e = row.endDate ? fmtDate(row.endDate) : null
      if (s && e) {
        return s === e ? s : `${s} — ${e}`
      }
      return s || e || '-'
    },
    formatDateTime(val) {
      return fmtDateTime(val)
    },
    // Ações de cliente (exemplos - ajuste endpoints conforme sua API)
    previewMessage(event, client) {
      console.log('Prévia de mensagem', { event, client })
      alert(`Prévia:\nEvento: ${event?.eventId}\nCliente: ${client?.name}\nNúmero: ${client?.phone}`)
    },
    async sendNow(event, client) {
      try {
        await this.api.post(`/events/${encodeURIComponent(event.eventId)}/send`, {
          phone: client.phone
        })
        alert('Mensagem enviada (solicitada).')
      } catch (err) {
        console.error('Erro ao enviar agora:', err)
        alert('Falha ao solicitar envio.')
      }
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
/* Layout base */
.events-page {
  padding: var(--space-xl, 24px);
}

/* Estado vazio */
.empty-state {
  background: var(--s-2, #fff);
  border-radius: var(--border-radius-large, 12px);
  padding: var(--space-xl, 24px);
  text-align: center;
  box-shadow: var(--shadow-sm, 0 1px 3px rgba(0,0,0,.08));
  max-width: 840px;
  margin: 10vh auto;
}
.empty-state h2 {
  margin: 0 0 8px;
}
.empty-state p {
  margin: 0 0 16px;
  color: var(--s-10, #666);
}
.empty-state .btn + .btn {
  margin-left: 8px;
}

/* Toolbar e título */
.grid-title h2 {
  margin: 0;
  font-size: 18px;
}

.legend {
  display: inline-flex;
  align-items: center;
  gap: 12px;
  margin-right: 8px;
  color: var(--s-10, #666);
  font-size: 12px;
}
.legend__dot {
  display: inline-block;
  width: 10px; height: 10px;
  border-radius: 50%;
  margin-right: 6px;
}
.legend__dot--active   { background: #1a7f37; box-shadow: 0 0 0 2px rgba(26,127,55,.15); }
.legend__dot--inactive { background: #999;      box-shadow: 0 0 0 2px rgba(153,153,153,.15); }

/* Ações por linha */
.row-actions {
  display: inline-flex;
  gap: 8px;
}

/* Botões utilitários */
.btn { cursor: pointer; border: 0; border-radius: 10px; padding: 8px 12px; }
.btn--primary   { background: #2563eb; color: #fff; }
.btn--secondary { background: #e5e7eb; color: #111827; }
.btn--danger    { background: #dc2626; color: #fff; }
.btn--muted     { background: #e5e7eb; color: #374151; }
.btn--success   { background: #16a34a; color: #fff; }
.btn--small     { padding: 6px 10px; font-size: 12px; }
.btn--xsmall    { padding: 4px 8px; font-size: 11px; }
.btn--pill      { padding: 4px 10px; border-radius: 999px; }

/* Modal */
.form-modal {
  position: fixed; inset: 0;
  background: rgba(0,0,0,0.45);
  display: flex; align-items: center; justify-content: center;
  z-index: 900;
}
.form-modal__content {
  background: var(--s-2, #fff);
  padding: 20px 24px;
  border-radius: 14px;
  box-shadow: 0 10px 30px rgba(0,0,0,.16);
  width: 100%; max-width: 920px; max-height: 90vh; overflow: auto;
  position: relative;
  animation: modalIn .16s ease-out;
}
.form-modal__close {
  position: absolute; top: 10px; right: 10px;
}

/* Grids */
.events-grid { max-width: 1200px; margin: 0 auto; }
.clients-grid { margin: 8px 0; }

@keyframes modalIn {
  from { transform: translateY(6px); opacity: 0; }
  to   { transform: translateY(0);   opacity: 1; }
}
</style>
