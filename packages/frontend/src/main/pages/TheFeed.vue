<template>
  <div>
    <portal v-if="canRenderToolbarPortal" to="toolbar">
      <div class="font-weight-bold">Feed</div>
    </portal>
    <v-row>
      <v-col cols="12" lg="12">
        <feed-timeline />
      </v-col>
    </v-row>
    <latest-blogposts />
  </div>
</template>
<script>
import {
  STANDARD_PORTAL_KEYS,
  buildPortalStateMixin
} from '@/main/utils/portalStateManager'
import FeedTimeline from '@/main/components/feed/FeedTimeline.vue'
import LatestBlogposts from '@/main/components/feed/LatestBlogposts.vue'

export default {
  name: 'TheFeed',
  components: {
    FeedTimeline,
    LatestBlogposts
  },
  mixins: [buildPortalStateMixin([STANDARD_PORTAL_KEYS.Toolbar], 'feed', 0)]
}
</script>
