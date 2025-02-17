<template>
  <b-navbar
    fixed-top
    spaced
    wrapper-class="container"
    close-on-click
    mobile-burger
    :active.sync="isBurgerMenuOpened"
    :class="{ 'navbar-shrink': !showTopNavbar }">
    <template #brand>
      <b-navbar-item tag="nuxt-link" :to="{ path: '/' }" class="logo">
        <img
          src="~/assets/Koda_Beta.svg"
          alt="First NFT market explorer on Kusama and Polkadot"
          width="130"
          height="35" />
      </b-navbar-item>
      <div
        class="is-hidden-desktop is-flex is-flex-grow-1 is-align-items-center is-justify-content-flex-end"
        @click="closeBurgerMenu">
        <HistoryBrowser />
        <b-button
          v-if="!isRedesignedLandingPage"
          type="is-primary is-bordered-light ml-2"
          class="navbar-link-background"
          icon-right="search"
          @click="showMobileSearchBar" />
        <Search
          ref="mobilSearchRef"
          hide-filter
          class="mt-5 search-navbar-container-mobile" />
      </div>
    </template>
    <template #start>
      <Search
        v-if="!isRedesignedLandingPage"
        hide-filter
        class="search-navbar is-flex-grow-1 pb-0 is-hidden-touch"
        search-column-class="is-flex-grow-1" />
    </template>
    <template v-if="showTopNavbar || isBurgerMenuOpened" #end>
      <LazyHistoryBrowser
        id="NavHistoryBrowser"
        class="custom-navbar-item navbar-link-background is-hidden-touch" />
      <b-navbar-dropdown
        v-show="isCreateVisible"
        id="NavCreate"
        arrowless
        collapsible
        data-cy="create-dropdown">
        <template #label>
          <span>{{ $t('create') }}</span>
        </template>
        <b-tooltip
          label="Start by creating your collection and add NFTs to it"
          position="is-right">
          <b-navbar-item
            tag="nuxt-link"
            :to="`/${urlPrefix}/create`"
            data-cy="classic">
            {{ $t('classic') }}
          </b-navbar-item>
        </b-tooltip>
        <template v-if="isRmrk">
          <b-tooltip
            label="Simplified process to create your NFT in a single step"
            position="is-right"
            style="display: block">
            <b-navbar-item
              tag="nuxt-link"
              :to="`/${urlPrefix}/mint`"
              data-cy="simple">
              {{ $t('simple') }}
            </b-navbar-item>
          </b-tooltip>
          <b-tooltip
            label="AI powered process to create your NFT"
            position="is-right"
            append-to-body>
            <b-navbar-item
              tag="nuxt-link"
              :to="`/${urlPrefix}/creative`"
              data-cy="creative">
              {{ $t('creative') }}
            </b-navbar-item>
          </b-tooltip>
        </template>
      </b-navbar-dropdown>
      <b-navbar-item
        tag="nuxt-link"
        :to="`/${urlPrefix}/explore`"
        data-cy="explore">
        <span>{{ $t('explore') }}</span>
      </b-navbar-item>
      <b-navbar-dropdown
        v-if="isBsx || isSnek"
        id="NavStats"
        arrowless
        collapsible
        data-cy="stats">
        <template #label>
          <span>{{ $t('stats') }}</span>
        </template>
        <b-navbar-item
          tag="nuxt-link"
          :to="`${
            accountId
              ? `/${urlPrefix}/offers?target=${accountId}`
              : `/${urlPrefix}/offers`
          }`"
          data-cy="global-offers">
          {{ $t('navbar.globalOffers') }}
        </b-navbar-item>
        <b-navbar-item
          tag="nuxt-link"
          :to="`/${urlPrefix}/stats`"
          data-cy="offers-stats">
          <span> {{ $t('navbar.offerStats') }}</span>
        </b-navbar-item>
      </b-navbar-dropdown>
      <b-navbar-dropdown
        v-if="isRmrk"
        id="NavStats"
        arrowless
        collapsible
        data-cy="stats">
        <template #label>
          <span>{{ $t('stats') }}</span>
        </template>
        <template>
          <b-navbar-item tag="nuxt-link" to="/spotlight" data-cy="spotlight">
            {{ $t('spotlight.page') }}
          </b-navbar-item>
          <b-navbar-item
            tag="nuxt-link"
            to="/series-insight"
            data-cy="series-insight">
            Series
          </b-navbar-item>
          <b-navbar-item tag="nuxt-link" to="/sales" data-cy="sales">
            Sales
          </b-navbar-item>
          <b-navbar-item tag="nuxt-link" to="/hot" data-cy="hot">
            Hot
          </b-navbar-item>
        </template>
      </b-navbar-dropdown>
      <LazyChainSelect
        id="NavChainSelect"
        class="navbar-item has-dropdown"
        data-cy="chain-select" />
      <LazySwitchLocale
        id="NavLocaleChanger"
        class="navbar-item has-dropdown"
        data-cy="localChanger" />
      <ColorModeButton />
      <NavbarProfileDropdown
        id="NavProfile"
        :is-rmrk="isRmrk"
        :show-incomming-offers="isBsx || isSnek"
        :is-snek="isSnek"
        data-cy="profileDropdown" />
    </template>
    <template v-else #end>
      <div class="image is-32x32 mr-2">
        <BasicImage
          v-show="inCollectionPage && currentCollection.image"
          :src="currentCollection.image"
          :alt="navBarTitle"
          rounded />
      </div>
      <div class="title is-4">{{ navBarTitle }}</div>
    </template>
  </b-navbar>
</template>

<script lang="ts">
import { Component, Ref, mixins } from 'nuxt-property-decorator'
import { get } from 'idb-keyval'

import BasicImage from '@/components/shared/view/BasicImage.vue'
import ColorModeButton from '@/components/common/ColorModeButton.vue'
import Identity from '@/components/identity/IdentityIndex.vue'
import NavbarProfileDropdown from '@/components/rmrk/Profile/NavbarProfileDropdown.vue'
import Search from '@/components/search/Search.vue'

import PrefixMixin from '@/utils/mixins/prefixMixin'

import { createVisible } from '@/utils/config/permision.config'
import { identityStore } from '@/utils/idbStore'
import AuthMixin from '~~/utils/mixins/authMixin'
import ExperimentMixin from '~~/utils/mixins/experimentMixin'

@Component({
  components: {
    NavbarProfileDropdown,
    Search,
    Identity,
    BasicImage,
    ColorModeButton,
  },
})
export default class NavbarMenu extends mixins(
  PrefixMixin,
  AuthMixin,
  ExperimentMixin
) {
  protected showTopNavbar = true
  private isGallery: boolean = this.$route.path.includes('tab=GALLERY')
  private fixedTitleNavAppearDistance = 200
  private lastScrollPosition = 0
  private artistName = ''
  private isBurgerMenuOpened = false
  @Ref('mobilSearchRef') readonly mobilSearchRef

  get isRmrk(): boolean {
    return this.urlPrefix === 'rmrk' || this.urlPrefix === 'westend'
  }

  get isBsx(): boolean {
    return this.urlPrefix === 'bsx'
  }

  get isSnek(): boolean {
    return this.urlPrefix === 'snek' || this.urlPrefix === 'bsx'
  }

  get inCollectionPage(): boolean {
    return this.$route.name === 'rmrk-collection-id'
  }
  get inGalleryDetailPage(): boolean {
    return this.$route.name === 'rmrk-gallery-id'
  }
  get inUserProfilePage(): boolean {
    return this.$route.name === 'rmrk-u-id'
  }

  get isCreateVisible(): boolean {
    return createVisible(this.urlPrefix)
  }

  get isTargetPage(): boolean {
    // why?
    return (
      this.inCollectionPage ||
      this.inGalleryDetailPage ||
      this.inUserProfilePage
    )
  }
  get currentCollection() {
    return this.$store.getters['history/getCurrentlyViewedCollection'] || {}
  }
  get currentGalleryItemName() {
    return this.$store.getters['history/getCurrentlyViewedItem']?.name || ''
  }

  get isRedesignedLandingPage() {
    return this.$route.name === 'index' && this.redesign
  }

  get navBarTitle(): string {
    let title = ''
    if (this.inCollectionPage) {
      title = this.currentCollection.name
    } else if (this.inGalleryDetailPage) {
      title = this.currentGalleryItemName
    } else if (this.inUserProfilePage) {
      const address = this.$route.params.id
      title = this.artistName ? this.artistName : address
      if (!this.artistName) {
        this.fetchArtistIdentity(address)
      }
    }
    return title
  }

  async fetchArtistIdentity(address) {
    const identity = await get(address, identityStore)
    if (identity && identity.display) {
      this.artistName = identity.display
    }
  }

  onScroll() {
    const currentScrollPosition = document.documentElement.scrollTop
    if (currentScrollPosition <= 0) {
      this.showTopNavbar = true
      return
    }
    if (Math.abs(currentScrollPosition - this.lastScrollPosition) < 30) {
      return
    }
    this.showTopNavbar =
      currentScrollPosition < this.fixedTitleNavAppearDistance
    this.lastScrollPosition = currentScrollPosition
  }

  showMobileSearchBar() {
    this.mobilSearchRef?.focusInput()
  }

  closeBurgerMenu() {
    if (this.isBurgerMenuOpened) {
      this.isBurgerMenuOpened = false
    }
  }

  mounted() {
    window.addEventListener('scroll', this.onScroll)
  }

  beforeDestroy() {
    window.removeEventListener('scroll', this.onScroll)
  }
}
</script>
