<template>
  <div class="contact">
    <v-row class="text-center pa-3">
      <v-col
        cols="12"
      >
        <h1>お問い合わせ</h1>
        <v-text-field
          v-model="title"
          label="件名"
        />
        <v-text-field
          v-model="contact"
          label="ご連絡先"
        />
        <v-textarea
          v-model="content"
          label="内容"
        />
      </v-col>
    </v-row>
  </div>
</template>
<script>
export default {
  name: 'Contact',
  data () {
    return {
      title: null,
      content: null,
      contact: null
    }
  }

}
</script>
<style lang="stylus" scoped>

</style>