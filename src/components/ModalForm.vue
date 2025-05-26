<template>
    <div v-if="isModalOpen" class="modal show d-block" tabindex="-1" style="background-color: rgba(0, 0, 0, 0.5)">
      <div class="modal-dialog">
        <div class="modal-content">
          <!-- Header -->
          <div class="modal-header">
            <h5 class="modal-title">{{ modalTitle }}</h5>
            <button type="button" class="btn-close" @click="$emit('close-modal')" aria-label="Close"></button>
          </div>
  
          <!-- Body -->
          <div class="modal-body">
            <slot />
          </div>
  
          <!-- Footer -->
          <div class="modal-footer">
            <slot name="footer">
              <button type="button" class="btn btn-secondary" @click="$emit('close-modal')">Close</button>
            </slot>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    name: "ModalFormComponent",
    props: {
      isModalOpen: {
        type: Boolean,
        default: false,
      },
      modalTitle: {
        type: String,
        required: true,
      },
    },
  };
  </script>
  