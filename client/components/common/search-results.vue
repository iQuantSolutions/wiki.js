<template lang="pug">
.search-results(v-if="searchIsFocused || (search && search.length > 1)")
  .search-results-container
    .search-results-help(v-if="!search || (search && search.length < 2)")
      .mt-4 {{ $t("common:header.searchHint") }}
    .search-results-loader(
      v-else-if="searchIsLoading && (!results || results.length < 1)"
    )
      orbit-spinner(:animation-duration="1000", :size="100", color="#FFF")
      .headline.mt-5 {{ $t("common:header.searchLoading") }}
    .search-results-none(
      v-else-if="!searchIsLoading && (!results || results.length < 1)"
    )
      img.mb-4(src="/_assets/svg/exclamation-triangle.svg", alt="No Results")
      .subheading {{ $t("common:header.searchNoResult") }}
    template(
      v-if="search && search.length >= 2 && results && results.length > 0"
    )
      v-subheader.ps-0 {{ $t("common:header.searchResultsCount", { total: response.totalHits }) }}
      v-list.search-results-items.radius-7.py-0(two-line, dense)
        template(v-for="(item, idx) of results")
          v-list-item(
            @click="goToPage(item)",
            @click.middle="goToPageInNewTab(item)",
            :key="item.id",
            :class="idx === cursor ? `highlighted` : ``"
          )
            v-list-item-content
              v-list-item-title(v-text="item.title")
              v-list-item-subtitle.caption(v-text="item.description")
              .caption.grey--text(v-text="item.path")
            v-list-item-action
              v-chip(label, outlined) {{ item.locale.toUpperCase() }}
          v-divider(v-if="idx < results.length - 1")
      v-pagination.mt-3(
        v-if="paginationLength > 1",
        dark,
        v-model="pagination",
        :length="paginationLength",
        circle
      )
    template(v-if="suggestions && suggestions.length > 0")
      v-subheader.mt-3 {{ $t("common:header.searchDidYouMean") }}
      v-list.search-results-suggestions.radius-7(dense, dark)
        template(v-for="(term, idx) of suggestions")
          v-list-item(
            :key="term",
            @click="setSearchTerm(term)",
            :class="idx + results.length === cursor ? `highlighted` : ``"
          )
            v-list-item-avatar
              v-icon mdi-magnify
            v-list-item-content
              v-list-item-title(v-text="term")
          v-divider(v-if="idx < suggestions.length - 1")
    .text-center.pt-5(v-if="search && search.length > 1")
      //- v-btn.mx-2(outlined, color='orange', @click='search = ``', v-if='results.length > 0')
      //-   v-icon(left) mdi-content-save
      //-   span {{$t('common:header.searchCopyLink')}}
      v-btn.home-btn(@click="search = ``")
        span {{ $t("common:header.searchClose") }}
</template>

<script>
import _ from 'lodash'
import { sync } from 'vuex-pathify'
import { OrbitSpinner } from 'epic-spinners'

import searchPagesQuery from 'gql/common/common-pages-query-search.gql'

export default {
  components: {
    OrbitSpinner
  },
  data() {
    return {
      cursor: 0,
      pagination: 1,
      perPage: 10,
      response: {
        results: [],
        suggestions: [],
        totalHits: 0
      }
    }
  },
  computed: {
    search: sync('site/search'),
    searchIsFocused: sync('site/searchIsFocused'),
    searchIsLoading: sync('site/searchIsLoading'),
    searchRestrictLocale: sync('site/searchRestrictLocale'),
    searchRestrictPath: sync('site/searchRestrictPath'),
    results() {
      const currentIndex = (this.pagination - 1) * this.perPage
      return this.response.results ?
        _.slice(
          this.response.results,
          currentIndex,
          currentIndex + this.perPage
        ) :
        []
    },
    hits() {
      return this.response.totalHits ? this.response.totalHits : 0
    },
    suggestions() {
      return this.response.suggestions ? this.response.suggestions : []
    },
    paginationLength() {
      return this.response.totalHits > 0 ?
        Math.ceil(this.response.totalHits / this.perPage) :
        0
    }
  },
  watch: {
    search(newValue, oldValue) {
      this.cursor = 0
      if (!newValue || (newValue && newValue.length < 2)) {
        this.searchIsLoading = false
      } else {
        this.searchIsLoading = true
      }
    },
    results() {
      this.cursor = 0
    }
  },
  mounted() {
    this.$root.$on('searchMove', (dir) => {
      this.cursor += dir === 'up' ? -1 : 1
      if (this.cursor < -1) {
        this.cursor = -1
      } else if (
        this.cursor >
        this.results.length + this.suggestions.length - 1
      ) {
        this.cursor = this.results.length + this.suggestions.length - 1
      }
    })
    this.$root.$on('searchEnter', () => {
      if (!this.results) {
        return
      }

      if (this.cursor >= 0 && this.cursor < this.results.length) {
        this.goToPage(_.nth(this.results, this.cursor))
      } else if (this.cursor >= 0) {
        this.setSearchTerm(
          _.nth(this.suggestions, this.cursor - this.results.length)
        )
      }
    })
  },
  methods: {
    setSearchTerm(term) {
      this.search = term
    },
    goToPage(item) {
      window.location.assign(`/${item.locale}/${item.path}`)
    },
    goToPageInNewTab(item) {
      window.open(`/${item.locale}/${item.path}`, '_blank')
    }
  },
  apollo: {
    response: {
      query: searchPagesQuery,
      variables() {
        return {
          query: this.search
        }
      },
      fetchPolicy: 'network-only',
      debounce: 300,
      throttle: 1000,
      skip() {
        return !this.search || this.search.length < 2
      },
      result() {
        this.pagination = 1
      },
      update: (data) => _.get(data, 'pages.search', {}),
      watchLoading(isLoading) {
        this.searchIsLoading = isLoading
      }
    }
  }
}
</script>

<style lang="scss">
.search-results {
  position: fixed;
  top: 64px;
  left: 0;
  overflow-y: auto;
  width: 100%;
  height: calc(100% - 64px);
  background-color: white;
  z-index: 100;
  text-align: center;
  animation: searchResultsReveal 0.3s ease;

  .v-subheader {
    color: #2d3748;
    font-weight: 500;
  }

  .v-application & .subheading {
    line-height: 100% !important;
  }

  img {
    height: 3rem;
    width: 3rem;
    z-index: 2;
  }

  .search-results-items {
    border: 1px solid rgba(0, 0, 0, 0.08);
    background-color: white !important;
    border-radius: 0.75rem !important;
    min-height: 3.5rem !important;

    .v-list-item {
      min-height: 3.5rem !important;
      padding-left: 0.75rem !important;
      padding-right: 0.75rem !important;

      &__content {
        padding: 0 !important;
      }

      &__title {
        font-size: 1rem !important;
        color: #2d3748 !important;
      }
    }

    .caption {
      color: #2d3748 !important;
      font-weight: 400 !important;
      letter-spacing: normal !important;
    }

    .highlighted {
      background: none !important;
    }

    .v-chip {
      padding-left: 0.5rem !important;
      padding-right: 0.5rem !important;
      border-radius: 0.5rem !important;
    }
  }

  .v-btn {
    z-index: 2;

    &.home-btn {
      border-radius: 0.5rem;
      border: 1px solid #e4ecf7 !important;
      background: #fff !important;
      display: inline-flex;
      padding: 0 !important;
      justify-content: center;
      align-items: center;
      text-transform: none;
      box-shadow: none !important;
    }
  }

  @media #{map-get($display-breakpoints, 'sm-and-down')} {
    top: 112px;
  }

  &-container {
    margin: 12px auto;
    max-width: 36.5rem;
  }

  &-help {
    text-align: center;
    font-size: 18px;
    font-weight: 400;
    color: #2d3748 !important;
  }

  &-loader {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    padding: 32px 0;
    color: #fff;
  }

  &-none {
    color: #fff;
  }

  &-items {
    text-align: left;

    .highlighted {
      background: #fff linear-gradient(to bottom, #fff, mc("orange", "100"));

      @at-root .theme--dark & {
        background: mc("grey", "900")
          linear-gradient(
            to bottom,
            mc("orange", "900"),
            darken(mc("orange", "900"), 15%)
          );
      }
    }
  }

  &-suggestions {
    .highlighted {
      background: transparent
        linear-gradient(to bottom, mc("blue", "500"), mc("blue", "700"));
    }
  }
}

@keyframes searchResultsReveal {
  0% {
    background-color: rgba(255, 255, 255, 0);
    padding-top: 0;
  }
  100% {
    background-color: rgba(255, 255, 255, 0.9);
    padding-top: 0;
  }
}
</style>
