<template>
  <div class="search-bar-container">
    <b-autocomplete
      ref="searchRef"
      v-model="name"
      class="gallery-search"
      :placeholder="$t('general.searchPlaceholder')"
      icon="search"
      :open-on-focus="showDefaultSuggestions"
      clearable
      max-height="500"
      dropdown-position="bottom"
      expanded
      @blur="$emit('blur')"
      @keydown.native.enter="onEnter">
      <template #header>
        <SearchSuggestion
          :name="name"
          :show-default-suggestions="showDefaultSuggestions"
          :query="query"
          @gotoGallery="redirectToGalleryPageIfNeed"
          @close="closeDropDown">
        </SearchSuggestion>
      </template>
    </b-autocomplete>
    <div class="search-bar-bg"></div>
    <img
      v-if="!name"
      class="search-bar-keyboard-icon"
      src="/search-k-keyboard.svg" />
  </div>
</template>

<script lang="ts">
import {
  Component,
  Emit,
  Prop,
  Ref,
  VModel,
  mixins,
} from 'nuxt-property-decorator'
import { SearchQuery } from './types'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import KeyboardEventsMixin from '~/utils/mixins/keyboardEventsMixin'

const SearchPageRoutePathList = ['/collections', '/gallery', '/explore']

@Component({
  components: {
    SearchSuggestion: () => import('./SearchSuggestion.vue'),
  },
})
export default class SearchBar extends mixins(
  PrefixMixin,
  KeyboardEventsMixin
) {
  @Prop({ type: Object, required: false }) public query!: SearchQuery
  @VModel({ type: String }) name!: string
  @Ref('searchRef') readonly searchRef

  public created() {
    this.initKeyboardEventHandler({
      k: this.bindSearchEvents,
    })
  }

  @Emit('enter')
  onEnter() {
    this.redirectToGalleryPageIfNeed()
    this.closeDropDown()
  }

  public focusInput(): void {
    this.searchRef?.focus()
  }

  private bindSearchEvents(event) {
    event.preventDefault()
    if (event.key === 'k') {
      this.focusInput()
    }
  }

  public closeDropDown() {
    this.searchRef.isActive = false
  }

  get showDefaultSuggestions() {
    return this.urlPrefix === 'rmrk'
  }

  redirectToGalleryPageIfNeed(params?: Record<string, string>) {
    if (SearchPageRoutePathList.indexOf(this.$route.path) === -1) {
      this.$router.replace({
        name: `${this.urlPrefix}-explore`,
        query: {
          ...this.$route.query,
          ...params,
        },
      })
    }
  }
}
</script>
