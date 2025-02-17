<template>
  <div class="search-suggestion-container" @click="resetSelectedIndex">
    <b-tabs
      v-show="name"
      v-model="activeSearchTab"
      destroy-on-hide
      expanded
      @input="resetSelectedIndex">
      <b-tab-item label="Collections" value="Collections">
        <div v-if="isCollectionResultLoading">
          <SearchResultItem
            v-for="item in searchSuggestionEachTypeMaxNum"
            :key="item"
            is-loading />
        </div>
        <div v-else>
          <div
            v-for="(item, idx) in collectionSuggestion"
            :key="item.id"
            :value="item"
            :class="`link-item ${idx === selectedIndex ? 'selected-item' : ''}`"
            @click="gotoCollectionItem(item)">
            <SearchResultItem :image="item.image">
              <template #content>
                <div
                  class="is-flex is-flex-direction-row is-justify-content-space-between pt-2 pr-2">
                  <span class="main-title name">{{ item.name }}</span>
                  <span>{{ urlPrefix.toUpperCase() }}</span>
                </div>
              </template>
            </SearchResultItem>
          </div>
        </div>
        <nuxt-link
          :to="{
            name: routeOf('explore'),
            query: { ...$route.query, tab: 'COLLECTION' },
          }"
          @click.native="$emit('close')">
          <div :class="loadMoreItemClassName">
            {{ $t('search.seeAll') }} -->
          </div>
        </nuxt-link>
      </b-tab-item>
      <b-tab-item label="NFTs" value="NFTs">
        <div v-if="isNFTResultLoading">
          <SearchResultItem
            v-for="item in searchSuggestionEachTypeMaxNum"
            :key="item"
            is-loading />
        </div>
        <div v-else>
          <div
            v-for="(item, idx) in nftSuggestion"
            :key="item.id"
            :value="item"
            :class="`link-item ${idx === selectedIndex ? 'selected-item' : ''}`"
            @click="gotoGalleryItem(item)">
            <SearchResultItem :image="item.image">
              <template #content>
                <div
                  class="is-flex is-flex-direction-row is-justify-content-space-between pt-2 pr-2">
                  <span class="main-title name">{{ item.name }}</span>
                  <span>{{ urlPrefix.toUpperCase() }}</span>
                </div>
                <div
                  class="is-flex is-flex-direction-row is-justify-content-space-between pt-1 pr-2">
                  <span class="name">{{ item.collection?.name }}</span>
                  <span v-if="item.price && parseFloat(item.price) > 0">
                    {{ $t('offer.price') }}:
                    <Money :value="item.price" inline />
                  </span>
                </div>
              </template>
            </SearchResultItem>
          </div>
        </div>
        <nuxt-link
          :to="{
            name: routeOf('explore'),
            query: { ...$route.query, tab: 'GALLERY' },
          }"
          @click.native="$emit('close')">
          <div :class="loadMoreItemClassName">
            {{ $t('search.seeAll') }} -->
          </div>
        </nuxt-link>
      </b-tab-item>
      <b-tab-item disabled label="User" value="User"> </b-tab-item>
    </b-tabs>
    <b-tabs v-show="!name" v-model="activeTrendingTab" destroy-on-hide expanded>
      <b-tab-item label="Trending" value="Trending">
        <div
          v-for="(item, idx) in defaultCollectionSuggestions"
          :key="item.id"
          :value="item"
          :class="`link-item ${idx === selectedIndex ? 'selected-item' : ''}`"
          @click="gotoCollectionItem(item)">
          <SearchResultItem :image="item.image">
            <template #content>
              <div class="pt-2 pr-2">
                <span class="main-title name">{{ item.name }}</span>
              </div>
              <div
                class="is-flex is-flex-direction-row is-justify-content-space-between pt-1 pr-2">
                <span>{{ $t('search.units') }}: {{ item.total }}</span>
                <span
                  >{{ $t('search.owners') }}: {{ item.uniqueCollectors }}</span
                >
                <span>{{ urlPrefix.toUpperCase() }}</span>
              </div>
            </template>
          </SearchResultItem>
        </div>
        <nuxt-link
          :to="{ name: 'series-insight' }"
          @click.native="$emit('close')">
          <div :class="loadMoreItemClassName">
            {{ $t('search.rankings') }} -->
          </div>
        </nuxt-link>
      </b-tab-item>
    </b-tabs>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Prop, Watch, mixins } from 'nuxt-property-decorator'
import { Debounce } from 'vue-debounce-decorator'
import seriesInsightList from '@/queries/rmrk/subsquid/seriesInsightList.graphql'
import { SearchQuery } from './types'
import { denyList } from '@/utils/constants'
import {
  CollectionWithMeta,
  NFTWithMeta,
} from '@/components/rmrk/service/scheme'
import { getSanitizer } from '@/components/rmrk/utils'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import { logError, mapNFTorCollectionMetadata } from '~/utils/mappers'
import {
  getCloudflareImageLinks,
  processMetadata,
} from '~/utils/cachingStrategy'
import { fastExtract } from '~/utils/ipfs'
import resolveQueryPath from '@/utils/queryPathResolver'
import { unwrapSafe } from '~/utils/uniquery'
import { RowSeries } from '~/components/series/types'

@Component({
  components: {
    Money: () => import('@/components/shared/format/Money.vue'),
  },
})
export default class SearchSuggestion extends mixins(PrefixMixin) {
  @Prop(String) public name!: string
  @Prop(Boolean) public showDefaultSuggestions!: boolean
  @Prop({ type: Object, required: false }) public query!: SearchQuery
  public searchSuggestionEachTypeMaxNum = 5
  public defaultCollectionSuggestions: (CollectionWithMeta & RowSeries)[] = []
  public activeSearchTab = 'Collections'
  public activeTrendingTab = 'Trending'
  public selectedIndex = -1
  public isCollectionResultLoading = false
  public isNFTResultLoading = false
  private nftResult: NFTWithMeta[] = []
  private collectionResult: CollectionWithMeta[] = []

  fetch() {
    this.fetchSuggestions()
  }

  public created() {
    document.addEventListener('keydown', this.onKeyDown)
    document.addEventListener('click', this.resetSelectedIndex)
  }

  beforeDestroy() {
    document.removeEventListener('keydown', this.onKeyDown)
    document.removeEventListener('click', this.resetSelectedIndex)
  }

  private onKeyDown(event: KeyboardEvent) {
    switch (event.key) {
      case 'ArrowUp':
        this.onKeydownSelected(-1)
        break
      case 'ArrowDown':
        this.onKeydownSelected(+1)
        break
      case 'Enter':
        this.nativeSearch()
        break
    }
  }

  get totalItemsAtCurrentTab() {
    if (!this.name) {
      return this.defaultCollectionSuggestions.length
    }
    return this.activeSearchTab === 'NFTs'
      ? this.nftSuggestion.length
      : this.collectionSuggestion.length
  }

  get collectionSuggestion() {
    return this.collectionResult.slice(0, this.searchSuggestionEachTypeMaxNum)
  }

  get nftSuggestion() {
    return this.nftResult.slice(0, this.searchSuggestionEachTypeMaxNum)
  }

  get loadMoreItemClassName() {
    return this.selectedIndex === this.totalItemsAtCurrentTab
      ? 'selected-item'
      : 'link-item'
  }

  get queryVariables() {
    return {
      first: this.searchSuggestionEachTypeMaxNum,
      offset: 0,
      denyList,
      orderBy: this.query.sortByMultiple?.length
        ? this.query.sortByMultiple
        : undefined,
      search: this.buildSearchParam(),
    }
  }

  get selectedItemListMap() {
    return {
      Trending: this.defaultCollectionSuggestions,
      Collections: this.collectionSuggestion,
      NFTs: this.nftSuggestion,
    }
  }

  public async fetchSuggestions() {
    if (this.showDefaultSuggestions) {
      try {
        const result = await this.$apollo.query({
          query: seriesInsightList,
          client: this.client,
          variables: {
            limit: this.searchSuggestionEachTypeMaxNum,
            orderBy: 'volume_DESC',
          },
        })

        const {
          data: { collectionEntities: collections },
        } = result

        const collectionMetadataList = collections
          .slice(0, this.searchSuggestionEachTypeMaxNum)
          .map(mapNFTorCollectionMetadata)
        getCloudflareImageLinks(collectionMetadataList).then((imageLinks) => {
          const collectionResult: (CollectionWithMeta & RowSeries)[] = []
          processMetadata<CollectionWithMeta>(
            collectionMetadataList,
            (meta, i) => {
              collectionResult.push({
                ...collections[i],
                ...meta,
                image:
                  (collections[i]?.metadata &&
                    imageLinks[fastExtract(collections[i].metadata)]) ||
                  getSanitizer(meta.image || '')(meta.image || ''),
              })
            }
          ).then(() => {
            this.defaultCollectionSuggestions = collectionResult
          })
        })
      } catch (e) {
        this.$consola.warn(e, 'Error while fetching default suggestions')
      }
    }
  }

  nativeSearch() {
    // not selected
    if (this.selectedIndex === -1) {
      return
    }

    const isSeeMore = this.selectedIndex >= this.totalItemsAtCurrentTab
    // trending collection
    if (!this.name) {
      if (isSeeMore) {
        this.$router.push({ name: 'series-insight' })
      } else {
        this.gotoCollectionItem(
          this.selectedItemListMap['Trending'][this.selectedIndex]
        )
      }
      return
    }

    // search result
    if (isSeeMore) {
      this.$emit('gotoGallery', {
        tab: this.activeSearchTab === 'Collections' ? 'COLLECTION' : 'GALLERY',
      })
    } else {
      if (this.activeSearchTab === 'NFTs') {
        this.gotoGalleryItem(
          this.selectedItemListMap['NFTs'][this.selectedIndex]
        )
      } else {
        this.gotoCollectionItem(
          this.selectedItemListMap['Collections'][this.selectedIndex]
        )
      }
    }
  }

  onKeydownSelected(step: 1 | -1) {
    const total = this.totalItemsAtCurrentTab + 1
    this.selectedIndex = (total + this.selectedIndex + step) % total
  }

  resetSelectedIndex() {
    this.selectedIndex = -1
  }

  @Emit('close')
  public gotoGalleryItem(item: NFTWithMeta) {
    this.$router.push(`/${this.urlPrefix}/gallery/${item.id}`)
  }

  @Emit('close')
  public gotoCollectionItem(item: CollectionWithMeta) {
    this.$router.push(`/${this.urlPrefix}/collection/${item.id}`)
  }

  private buildSearchParam(): Record<string, unknown>[] {
    const params: any[] = []

    if (this.query.search) {
      params.push({ name_containsInsensitive: this.query.search })
    }

    if (this.query.listed) {
      params.push({ price_gt: '0' })
    }

    return params
  }

  public routeOf(url: string): string {
    return `${this.urlPrefix}-${url}`
  }

  @Debounce(50)
  async updateSuggestion(value: string) {
    //To handle empty string
    if (!value) {
      // reset query-based search results once searchString is empty
      this.collectionResult = []
      this.nftResult = []
      return
    }

    this.isCollectionResultLoading = true
    this.isNFTResultLoading = true
    this.query.search = value
    try {
      const queryNft = await resolveQueryPath(this.client, 'nftListWithSearch')
      const nfts = this.$apollo.query({
        query: queryNft.default,
        client: this.client,
        variables: this.queryVariables,
      })

      const {
        data: { nFTEntities },
      } = await nfts
      const nftList = unwrapSafe(
        nFTEntities.slice(0, this.searchSuggestionEachTypeMaxNum)
      )
      const metadataList: string[] = nftList.map(mapNFTorCollectionMetadata)
      getCloudflareImageLinks(metadataList).then((imageLinks) => {
        const nftResult: NFTWithMeta[] = []
        processMetadata<NFTWithMeta>(metadataList, (meta, i) => {
          nftResult.push({
            ...nftList[i],
            ...meta,
            image:
              (nftList[i]?.metadata &&
                imageLinks[fastExtract(nftList[i].metadata)]) ||
              getSanitizer(meta.image || '')(meta.image || ''),
            animation_url: getSanitizer(meta.animation_url || '')(
              meta.animation_url || ''
            ),
          })
        }).then(() => {
          this.nftResult = nftResult
          this.isNFTResultLoading = false
        })
      })
    } catch (e) {
      logError(e, (msg) =>
        this.$consola.warn('[PREFETCH] Unable fo fetch', msg)
      )
      this.isNFTResultLoading = false
    }
    try {
      const query = await resolveQueryPath(
        this.client,
        'collectionListWithSearch'
      )

      const collectionResult = this.$apollo.query({
        query: query.default,
        client: this.client,
        variables: this.queryVariables,
      })

      const {
        data: { collectionEntities },
      } = await collectionResult
      const collections = unwrapSafe(
        collectionEntities.slice(0, this.searchSuggestionEachTypeMaxNum)
      )
      const metadataList: string[] = collections.map(mapNFTorCollectionMetadata)
      getCloudflareImageLinks(metadataList).then((imageLinks) => {
        const collectionResult: CollectionWithMeta[] = []
        processMetadata<CollectionWithMeta>(metadataList, (meta, i) => {
          collectionResult.push({
            ...collections[i],
            ...meta,
            image:
              (collections[i]?.metadata &&
                imageLinks[fastExtract(collections[i].metadata)]) ||
              getSanitizer(meta.image || '')(meta.image || ''),
          })
        }).then(() => {
          this.collectionResult = collectionResult
          this.isCollectionResultLoading = false
        })
      })
    } catch (e) {
      logError(e, (msg) =>
        this.$consola.warn('[PREFETCH] Unable fo fetch', msg)
      )
      this.isCollectionResultLoading = false
    }
  }

  @Watch('name')
  onValueChange(value) {
    this.updateSuggestion(value)
    this.resetSelectedIndex()
  }
}
</script>
