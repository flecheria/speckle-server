<template>
  <v-list dense nav class="mt-0 py-0 mb-3">
    <v-list-item
      :class="`px-2 list-overlay-${$vuetify.theme.dark ? 'dark' : 'light'} elevation-2`"
      :style="`${stickyTop ? 'position: sticky; top: 82px;' : ''}`"
      @click="expand = !expand"
    >
      <v-list-item-action>
        <v-icon small>mdi-filter-variant</v-icon>
      </v-list-item-action>
      <v-list-item-content>
        <v-list-item-title>
          <span v-if="activeFilter === null">
            Filters
            <span class="caption grey--text">({{ allFilters.length }})</span>
          </span>
          <span v-else>{{ activeFilter.name }}</span>
        </v-list-item-title>
      </v-list-item-content>
      <portal-target name="filter-actions"></portal-target>
      <v-list-item-action v-if="activeFilter" class="pa-0 ma-0">
        <v-btn
          v-tooltip="'Remove filter'"
          small
          icon
          @click.stop="
            activeFilter = null
            filterSearch = null
          "
        >
          <v-icon small>mdi-close</v-icon>
        </v-btn>
      </v-list-item-action>
      <v-list-item-action class="pa-0 ma-0">
        <v-btn small icon @click.stop="expand = !expand">
          <v-icon>{{ expand ? 'mdi-chevron-up' : 'mdi-chevron-down' }}</v-icon>
        </v-btn>
      </v-list-item-action>
    </v-list-item>
    <v-scroll-y-transition>
      <div v-show="expand">
        <div v-if="activeFilter && activeFilter.data.type === 'string'">
          <filter-category-active :filter="activeFilter" />
        </div>
        <div v-if="activeFilter && activeFilter.data.type === 'number'">
          <filter-numeric-active
            :filter="activeFilter"
            :prevent-first-set="preventFirstSet"
          />
        </div>
        <div v-show="activeFilter === null">
          <div class="">
            <v-text-field
              v-model="filterSearch"
              solo
              dense
              placeholder="Search all filters"
              append-icon="mdi-magnify"
              hide-details
              class="my-2"
              :style="`${stickyTop ? 'position: sticky; top: 128px;' : ''} z-index: 6`"
            />
            <div v-if="topFilters.length !== 0 && !filterSearch">
              <v-subheader>Recommended filters:</v-subheader>
              <div v-for="(filter, index) in topFilters" :key="index">
                <filter-row-select
                  v-if="filter"
                  :filter="filter"
                  @active-toggle="(e) => (activeFilter = e)"
                />
              </div>
            </div>
            <v-subheader>
              {{ filterSearch ? 'Matching' : 'Other' }} filters:
            </v-subheader>
            <div v-for="filter in matchingFilters" :key="filter.targetKey">
              <filter-row-select
                :filter="filter"
                @active-toggle="(e) => (activeFilter = e)"
              />
            </div>
          </div>
        </div>
      </div>
    </v-scroll-y-transition>
  </v-list>
</template>
<script>
export default {
  name: 'ViewerFilters',
  components: {
    FilterRowSelect: () => import('@/main/components/viewer/FilterRowSelect'),
    FilterCategoryActive: () => import('@/main/components/viewer/FilterCategoryActive'),
    FilterNumericActive: () => import('@/main/components/viewer/FilterNumericActive')
  },
  props: {
    props: {
      type: Object,
      default: () => {}
    },
    sourceApplication: {
      type: String,
      default: ''
    },
    stickyTop: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      expand: false,
      revitFilters: ['type', 'family', 'level'],
      allFilters: [],
      activeFilter: null,
      filterSearch: null,
      trySetPresetFilter: false,
      preventFirstSet: false
    }
  },
  computed: {
    topFilters() {
      if (this.allFilters.length === 0) return []
      const arr = []
      arr.push(this.allFilters.find((f) => f.name === 'Object Type'))
      if (this.sourceApplication.toLowerCase().includes('revit')) {
        arr.push(this.allFilters.find((f) => f.name === 'Level'))
        arr.push(this.allFilters.find((f) => f.name === 'family'))
        arr.push(this.allFilters.find((f) => f.name === 'type'))
        arr.push(this.allFilters.find((f) => f.name === 'Area'))
        arr.push(this.allFilters.find((f) => f.name === 'Volume'))
      }
      return arr.filter((el) => !!el)
    },
    matchingFilters() {
      if (this.filterSearch === null) return this.allFilters
      else {
        return this.allFilters.filter(
          (f) =>
            f.name.toLowerCase().includes(this.filterSearch.toLowerCase()) ||
            f.targetKey.toLowerCase().includes(this.filterSearch.toLowerCase())
        )
      }
    }
  },
  watch: {
    props(newVal) {
      if (newVal) {
        this.parseAndSetFilters()
      }
    },
    '$store.state.appliedFilter'() {
      if (this.trySetPresetFilter) return
      if (this.$store.state.appliedFilter && this.$store.state.appliedFilter.filterBy) {
        const key = Object.keys(this.$store.state.appliedFilter.filterBy)[0]
        const presetFilter = this.allFilters.find((f) => f.targetKey === key)
        if (presetFilter) this.activeFilter = presetFilter
        this.trySetPresetFilter = true
        this.preventFirstSet = true
      }
    }
  },
  mounted() {
    if (this.props) {
      this.parseAndSetFilters()
    }
    if (this.$eventHub) {
      this.$eventHub.$on('structure-filters', () => {
        this.activeFilter = null
      })
      this.$eventHub.$on('selection-filters', () => {
        this.activeFilter = null
      })
    }
  },
  methods: {
    parseAndSetFilters() {
      const keys = Object.keys(this.props)
      const filters = []
      for (const key of keys) {
        const filter = {}
        // Handle revit params
        if (key.startsWith('parameters.')) {
          if (key.endsWith('.value')) {
            filter.name = this.props[key.replace('.value', '.name')].allValues[0]
            filter.targetKey = key
            filter.data = this.props[key]
            filters.push(filter)
            continue
          } else {
            continue
          }
        }
        // Beautify level name
        if (key === 'level.name') {
          filter.name = 'Level'
          filter.targetKey = key
          filter.data = this.props[key]
          filters.push(filter)
          continue
        }
        // Beautify speckle type
        if (key === 'speckle_type') {
          filter.name = 'Object Type'
          filter.targetKey = key
          filter.data = this.props[key]
          filters.push(filter)
          continue
        }
        // Skip some
        if (
          key.endsWith('.units') ||
          key.endsWith('.speckle_type') ||
          key.includes('.parameters.') ||
          key.includes('level.') ||
          key.includes('renderMaterial') ||
          key.includes('.domain') ||
          key.includes('plane.') ||
          key.includes('baseLine') ||
          key.includes('referenceLine') ||
          key.includes('end.') ||
          key.includes('start.') ||
          key.includes('endPoint.') ||
          key.includes('midPoint.') ||
          key.includes('startPoint.') ||
          key.includes('startPoint.') ||
          key.includes('displayStyle') ||
          key.includes('displayValue') ||
          key.includes('displayMesh')
        ) {
          continue
        }

        filter.name = key
        filter.targetKey = key
        filter.data = this.props[key]
        filters.push(filter)
      }
      this.allFilters = filters
    }
  }
}
</script>
<style scoped>
.list-overlay-dark {
  background: rgba(40, 40, 40, 1);
  z-index: 5;
}
.list-overlay-light {
  background: rgba(235, 235, 235, 1);
  z-index: 5;
}
</style>
