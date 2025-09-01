<template>
  <div>
    <button class="btn btn--primary open-form-button" @click="showForm = true">
      Novo Evento
    </button>

    <div v-if="showForm" class="form-modal">
      <div class="form-modal__content">
        <button type="button" class="form-modal__close btn btn--secondary btn--small" @click="closeForm">Fechar</button>
        <EventForm />
      </div>
    </div>
  </div>
</template>

<script>
import EventForm from './components/EventForm.vue'

export default {
  name: 'App',
  components: { EventForm },
  data() {
    return {
      showForm: false
    }
  },
  methods: {
    closeForm() {
      this.showForm = false
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

