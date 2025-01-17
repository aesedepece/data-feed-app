<template>
  <div class="content">
    <div class="section-header">
      <nuxt-link class="back-to-list" :to="localePath('/')">
        <font-awesome-icon class="icon" icon="arrow-alt-circle-left" />
      </nuxt-link>
      <div class="title">
        <SvgIcon v-if="svgIcon" :name="svgIcon" />
        <p v-if="feed && feed.name" class="feed-name">
          {{ feed.name.toUpperCase() }}
        </p>
      </div>
      <Networks
        v-if="feed && feed.network"
        class="network-details"
        :network="feed.network"
        :color="feed.color"
      />
    </div>
    <Chart
      v-if="feed"
      class="chart"
      :data="chartData"
      :data-label="feed.label"
      :name="feedName"
      @change-range="updateQuery"
    />
    <Fieldset
      :title="$t('data_feed_details.contract_address')"
      class="contract-container"
    >
      <a :href="url" target="_blank" class="contract-address">
        {{ feedAddress }}
        <font-awesome-icon class="icon" icon="external-link-alt" />
      </a>
    </Fieldset>
    <TransactionsList
      v-if="transactions"
      class="transactions"
      :transactions="transactions"
    />
    <el-pagination
      v-if="feed && numberOfPages > 1"
      :small="small"
      class="pagination"
      layout="prev, pager, next"
      :pager-count="5"
      :page-count="numberOfPages"
      :current-page="currentPage"
      @current-change="handleCurrentChange"
    />
  </div>
</template>

<script>
import feed from '@/apollo/queries/feed.gql'
import requests from '@/apollo/queries/requests.gql'
import { getWitnetBlockExplorerLink } from '@/utils/getWitnetBlockExplorerLink'
import { CHART_RANGE } from '@/constants'
import { getTimestampByRange } from '@/utils/getTimestampByRange.js'

export default {
  apollo: {
    feed: {
      prefetch: true,
      query: feed,
      variables() {
        return {
          timestamp: this.timestamp,
          feedFullName: this.feedFullName,
        }
      },
      pollInterval: 60000,
    },
    requests: {
      prefetch: true,
      query: requests,
      variables() {
        return {
          feedFullName: this.feedFullName,
          page: this.currentPage,
          size: this.itemsPerPage,
        }
      },
      pollInterval: 60000,
    },
  },
  data() {
    return {
      ranges: CHART_RANGE,
      currentPage: 1,
      itemsPerPage: 25,
      range: 24,
      timestamp: getTimestampByRange(CHART_RANGE.w.value),
      feedFullName: this.$route.params.id,
    }
  },
  computed: {
    small() {
      return this.numberOfPages > 10
    },
    svgIcon() {
      return this.feed ? this.feed.name.split('/').join('') : ''
    },
    url() {
      return this.feed
        ? this.feed.blockExplorer.replace(`{address}`, this.feedAddress)
        : ''
    },
    numberOfPages() {
      return this.feed
        ? Math.ceil(this.feed.requests.length / this.itemsPerPage)
        : 0
    },
    feedName() {
      return this.feed ? this.feed.name.toUpperCase() : ''
    },
    feedAddress() {
      return this.feed ? this.feed.address : ''
    },
    chartData() {
      if (this.feed && this.feed.requests.length > 0) {
        return this.feed.requests
          .map((request) => {
            return {
              time: Number(request.timestamp),
              value:
                request.result.slice(0, -3) + '.' + request.result.slice(-3),
            }
          })
          .sort((t1, t2) => t1.time - t2.time)
      } else {
        return [{ time: 0, value: 0 }]
      }
    },
    transactions() {
      if (this.feed && this.requests && this.requests.length > 0) {
        return this.requests.map((request) => ({
          witnetLink: getWitnetBlockExplorerLink(request.drTxHash),
          drTxHash: request.drTxHash,
          data: {
            label: this.feed.label,
            value: request.result,
          },
          timestamp: request.timestamp,
        }))
      } else {
        return null
      }
    },
  },
  methods: {
    handleCurrentChange(val) {
      this.currentPage = val
    },
    updateQuery(val) {
      this.timestamp = getTimestampByRange(this.ranges[val].value)
    },
  },
}
</script>

<style lang="scss" scoped>
.content {
  display: grid;
  grid-template: max-content max-content max-content 1fr / 1fr;
  row-gap: 16px;
  .contract-container {
    margin-top: 150px;
    .contract-address {
      display: flex;
      align-items: center;
      line-break: loose;
      word-break: break-all;
      font-size: 24px;
      padding: 24px;
      cursor: pointer;
      color: var(--contract-address);
      .icon {
        margin-left: 16px;
      }
    }
  }
  .pagination {
    padding-bottom: 72px;
    justify-self: center;
  }
  .chart {
    margin-top: 24px;
    height: 400px;
  }
  .section-header {
    display: grid;
    grid-template: 1fr / 1fr 1fr 1fr;
    justify-items: center;
    align-items: flex-end;
    margin-top: 16px;
    width: 100%;
    .network-details {
      justify-self: flex-end;
    }
    .title {
      display: flex;
      flex-direction: column;
      align-items: center;
      .feed-name {
        font-weight: 600;
        margin-top: 8px;
      }
    }
    .back-to-list {
      justify-self: flex-start;
      .icon {
        font-size: 24px;
        color: var(--text);
      }
      &:hover {
        .icon {
          color: var(--text-hover);
        }
      }
    }
  }
}
@media (max-width: 1200px) {
  .content {
    .section-header {
      .back-to-list {
        margin-left: 16px;
      }
      .network {
        margin-right: 16px;
      }
    }
  }
}
</style>
