<template>
  <QPage class="stl-footer--bottom">
    <section class="hero hero__background-container">
      <div class="q-pa-md">
        <div class="row items-center">
          <div class="col-md-6">
            <q-img
              src="statics/images/banner-img.jpg"
            />
          </div>
          <div class="col-md-6">
            <div class="hero__background bg-primary-gradient" />
            <div class="hero__content text-center text-white">
              <slot name="heroContent" />
            </div>
          </div>
        </div>
      </div>
    </section>

    <slot />
  </QPage>
</template>

<style lang="stylus" scoped>
.hero__background-container
  position: relative

.hero__background
  position: absolute
  top: 0
  right: 0
  bottom: 0
  left: 0
  z-index: -1

.hero__content
  padding: $spaces.sm.x
  @media (min-width $breakpoint-sm-min)
    padding: $spaces.xl.y $spaces.xl.x
</style>
