<template>  
  <div class="sw-Modal">
    <div class="sw-Modal_Inner">
      <div class="sw-Modal_InnerHeader" :class="{ 'a': index > 0 }" @click="onClick(index < 10)" #header>
        <div class="sw-Modal_Title" v-dir="a & b">Delete?</div>
      </div>
      <div class="sw-Modal_Text">
        cancel
        or does not cancel
      </div>
      <div class="sw-Modal_Btns">
        <Button class="mr-1" title="cancel" @click="close()"></Button>
        <Button title="delete" @click="deleteData()"></Button>
      </div>
    </div>
  </div>
</template>

<script>
  import Button from 'components/buttons/Button'

  export default {
    components: {
      Button
    },
    props: {
    },
    data() {
      return {
      }
    },
    methods: {
      close() {
        this.$emit('close')
      },
      deleteData() {
        this.close()
      },
    }
  }
</script>
