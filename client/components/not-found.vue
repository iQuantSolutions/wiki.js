<template lang='pug'>
  v-app
    .notfound
      .notfound-content
        img(src='/_assets/svg/file-earmark-x.svg', alt='Not Found')
        .headline {{$t('notfound.title')}}
        .subheading This page does not exist.
        v-btn.home-btn(href='/', depressed)
          span {{$t('notfound.gohome')}}
</template>

<script>

export default {
  data() {
    return { }
  }
}
</script>

<style lang='scss'>

</style>
