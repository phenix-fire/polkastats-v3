<template>
  <div>
    <section>
      <b-container class="page-phragmen main py-5">
        <h1 class="mb-4">
          {{ $t("pages.election-prediction.title") }}
        </h1>
        <template v-if="phragmenEnabled">
          <p v-if="blockHeight && timestamp">
            {{ $t("pages.election-prediction.last_execution") }}
            <nuxt-link
              v-b-tooltip.hover
              :to="`/block?blockNumber=${phragmen.block_height}`"
              title="Check block information"
            >
              #{{ formatNumber(blockHeight) }}
            </nuxt-link>
            ({{ timestamp }})
          </p>
          <!-- Filter -->
          <b-row>
            <b-col lg="12" class="mb-4">
              <b-form-input
                id="filterInput"
                v-model="filter"
                type="search"
                :placeholder="
                  $t('pages.election-prediction.search_placeholder')
                "
              />
            </b-col>
          </b-row>
          <!-- Mobile sorting -->
          <div class="row d-block d-sm-block d-md-block d-lg-none d-xl-none">
            <b-col lg="6" class="my-1">
              <b-form-group
                :label="$t('pages.election-prediction.sort')"
                label-cols-sm="3"
                label-align-sm="right"
                label-size="sm"
                label-for="sortBySelect"
                class="mb-4"
              >
                <b-input-group size="sm">
                  <b-form-select
                    id="sortBySelect"
                    v-model="sortBy"
                    :options="sortOptions"
                    class="w-75"
                  >
                    <template v-slot:first>
                      <option value="">
                        -- none --
                      </option>
                    </template>
                  </b-form-select>
                  <b-form-select
                    v-model="sortDesc"
                    size="sm"
                    :disabled="!sortBy"
                    class="w-25"
                  >
                    <option :value="false">
                      Asc
                    </option>
                    <option :value="true">
                      Desc
                    </option>
                  </b-form-select>
                </b-input-group>
              </b-form-group>
            </b-col>
          </div>
          <JsonCSV
            :data="candidatesJSON"
            class="download-csv mb-2"
            :name="`polkastats.io_polkadot_phragmen_at_${blockHeight}.csv`"
          >
            <i class="fas fa-file-csv"></i>
            {{ $t("pages.accounts.download_csv") }}
          </JsonCSV>
          <!-- Table with sorting and pagination-->
          <div>
            <b-table
              id="candidates-table"
              striped
              stacked="md"
              :fields="fields"
              :items="candidates"
              :per-page="perPage"
              :current-page="currentPage"
              :sort-by.sync="sortBy"
              :sort-desc.sync="sortDesc"
              :filter="filter"
              :filter-included-fields="filterOn"
              @filtered="onFiltered"
            >
              <template v-slot:cell(rank)="data">
                <p class="text-right mb-0">
                  {{ data.item.rank }}
                </p>
              </template>
              <template v-slot:cell(pub_key_stash)="data">
                <div
                  class="d-block d-sm-block d-md-none d-lg-none d-xl-none text-center"
                >
                  <span
                    v-if="data.item.elected"
                    v-b-tooltip.hover
                    :title="$t('details.validator.elected_for_next_session')"
                    class="circle blue"
                  >
                    <i class="fa fa-chevron-right mr-1" aria-hidden="true"></i>
                  </span>
                  <div>
                    <Identicon
                      :key="data.item.pub_key_stash"
                      :value="data.item.pub_key_stash"
                      :size="80"
                      :theme="'polkadot'"
                    />
                  </div>
                  <h4 v-if="data.item.display_name" class="mt-2 mb-2">
                    {{ data.item.display_name }}
                  </h4>
                  <h4 v-else class="mt-2 mb-2">
                    {{ shortAddress(data.item.pub_key_stash) }}
                  </h4>
                  <p class="mt-2 mb-2 rank">rank #{{ data.item.rank }}</p>
                  <p
                    v-b-tooltip.hover
                    class="bonded mb-0"
                    title="$t('pages.election-prediction.total_stake')"
                  >
                    {{ formatAmount(data.item.stake_total) }}
                  </p>
                </div>
                <div class="d-none d-sm-none d-md-block d-lg-block d-xl-block">
                  <div class="d-inline-block">
                    <span
                      v-if="data.item.elected"
                      v-b-tooltip.hover
                      :title="$t('details.validator.elected_for_next_session')"
                      class="circle blue"
                    >
                      <i
                        class="fa fa-chevron-right mr-1"
                        aria-hidden="true"
                      ></i>
                    </span>
                    <Identicon
                      :key="data.item.pub_key_stash"
                      :value="data.item.pub_key_stash"
                      :size="20"
                      :theme="'polkadot'"
                    />
                  </div>
                  <span v-if="data.item.display_name">
                    {{ data.item.display_name }}
                  </span>
                  <span v-else>
                    {{ shortAddress(data.item.pub_key_stash) }}
                  </span>
                </div>
              </template>
              <template v-slot:cell(other_stake_count)="data">
                <p class="text-right mb-0">
                  {{ data.item.other_stake_count }}
                </p>
              </template>
              <template v-slot:cell(stake_total)="data">
                <p v-if="data.item.stake_total > 0" class="text-right mb-0">
                  {{ formatAmount(data.item.stake_total) }}
                </p>
              </template>
              <template v-slot:cell(favorite)="data">
                <p class="text-center mb-0">
                  <a
                    class="favorite"
                    @click="toggleFavorite(data.item.pub_key_stash)"
                  >
                    <i
                      v-if="data.item.favorite"
                      v-b-tooltip.hover
                      class="fas fa-star"
                      style="color: #f1bd23"
                      :title="
                        $t('pages.election-prediction.remove_from_favorites')
                      "
                    />
                    <i
                      v-else
                      v-b-tooltip.hover
                      class="fas fa-star"
                      style="color: #e6dfdf;"
                      :title="$t('pages.election-prediction.add_to_favorites')"
                    />
                  </a>
                </p>
              </template>
            </b-table>
            <div style="display: flex">
              <b-pagination
                v-model="currentPage"
                :total-rows="totalRows"
                :per-page="perPage"
                aria-controls="validators-table"
              />
              <b-button-group class="mx-4">
                <b-button
                  v-for="(item, index) in tableOptions"
                  :key="index"
                  @click="handleNumFields(item)"
                >
                  {{ item }}
                </b-button>
              </b-button-group>
            </div>
          </div>
        </template>
      </b-container>
    </section>
  </div>
</template>
<script>
import { mapMutations } from "vuex";
import bootstrap from "bootstrap";
import Identicon from "../components/identicon.vue";
import { isHex } from "@polkadot/util";
import BN from "bn.js";
import { paginationOptions, network } from "../polkastats.config.js";
import commonMixin from "../mixins/commonMixin.js";
import JsonCSV from "vue-json-csv";
import gql from "graphql-tag";

export default {
  components: {
    Identicon,
    JsonCSV
  },
  mixins: [commonMixin],
  data: function() {
    return {
      phragmenEnabled: network.phragmenEnabled,
      tableOptions: paginationOptions,
      perPage: localStorage.paginationOptions
        ? parseInt(localStorage.paginationOptions)
        : 10,
      currentPage: 1,
      sortBy: `favorite`,
      sortDesc: true,
      filter: null,
      filterOn: [],
      totalRows: 1,
      fields: [
        {
          key: "rank",
          label: "#",
          sortable: true,
          class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell`
        },
        {
          key: "pub_key_stash",
          label: this.$i18n.t("pages.election-prediction.candidate"),
          sortable: true,
          filterByFormatted: true
        },
        {
          key: "other_stake_count",
          label: this.$i18n.t("pages.election-prediction.voters"),
          sortable: true,
          class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell`
        },
        {
          key: "stake_total",
          label: this.$i18n.t("pages.election-prediction.total_stake"),
          sortable: true,
          class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell`
        },
        {
          key: "favorite",
          label: "⭐",
          sortable: true,
          class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell`
        }
      ],
      polling: null,
      favorites: [],
      phragmen: {
        timestamp: 0,
        block_height: 0,
        candidates: []
      },
      validatorCount: undefined
    };
  },
  computed: {
    candidates() {
      return this.phragmen.candidates.map(candidate => {
        if (this.getDisplayName(candidate.pub_key_stash)) {
          candidate.display_name = this.getDisplayName(candidate.pub_key_stash);
        } else {
          candidate.display_name = ``;
        }
        candidate.favorite = this.isFavorite(candidate.pub_key_stash);
        return candidate;
      });
    },
    candidatesJSON() {
      return this.candidates.map(candidate => {
        return {
          rank: candidate.rank,
          name: candidate.display_name,
          stash_account: candidate.pub_key_stash,
          total_stake: candidate.stake_total,
          voters: candidate.other_stake_count
        };
      });
    },
    identities() {
      return this.$store.state.identities.list;
    },
    timestamp() {
      var newDate = new Date();
      newDate.setTime(this.phragmen.timestamp);
      return this.phragmen.timestamp ? newDate.toUTCString() : ``;
    },
    blockHeight() {
      return this.phragmen.block_height
        ? this.formatNumber(this.phragmen.block_height)
        : ``;
    },
    sortOptions() {
      // Create an options list from our fields
      return this.fields
        .filter(f => f.sortable)
        .map(f => {
          return { text: f.label, value: f.key };
        });
    }
  },
  watch: {
    favorites: function(favorites) {
      this.$cookies.set("favorites", favorites, {
        path: "/",
        maxAge: 60 * 60 * 24 * 7
      });
    }
  },
  created: function() {
    var vm = this;

    // Get favorites from cookie
    if (this.$cookies.get("favorites")) {
      this.favorites = this.$cookies.get("favorites");
    }

    // Force update of identity list if empty
    if (this.$store.state.identities.list.length === 0) {
      vm.$store.dispatch("identities/update");
    }

    // Update data every 60 seconds
    this.polling = setInterval(() => {
      vm.$store.dispatch("identities/update");
      if (!this.filter) {
        this.totalRows = this.phragmen.candidates.length;
      }
    }, 60000);
  },
  beforeDestroy: function() {
    clearInterval(this.polling);
  },
  methods: {
    handleNumFields(num) {
      this.perPage = parseInt(num);
    },
    toggleFavorite(accountId) {
      if (this.favorites.indexOf(accountId) !== -1) {
        this.favorites.splice(this.favorites.indexOf(accountId), 1);
      } else {
        this.favorites.push(accountId);
      }
      return true;
    },
    isFavorite(accountId) {
      return this.favorites.includes(accountId);
    },
    onFiltered(filteredItems) {
      // Trigger pagination to update the number of buttons/pages due to filtering
      this.totalRows = filteredItems.length;
      this.currentPage = 1;
    },
    getDisplayName: function(accountId) {
      let identity = this.$store.state.identities.list.find(
        identity => identity.accountId === accountId
      );
      if (identity) {
        identity = identity.identity;
        if (
          identity.displayParent &&
          identity.displayParent !== `` &&
          identity.display &&
          identity.display !== ``
        ) {
          return `${identity.displayParent} / ${identity.display}`;
        } else {
          return identity.display;
        }
      }
      return ``;
    }
  },
  apollo: {
    $subscribe: {
      phragmen: {
        query: gql`
          subscription phragmen {
            phragmen(limit: 1, order_by: { block_height: desc }) {
              timestamp
              phragmen_json
              block_height
            }
          }
        `,
        skip() {
          return !this.validatorCount;
        },
        result({ data }) {
          const phragmenOutput = JSON.parse(data.phragmen[0].phragmen_json);
          let candidates = [];
          Object.entries(phragmenOutput.supports).forEach(
            ([pub_key_stash, candidate]) => {
              candidates.push({
                pub_key_stash,
                other_stake_count: candidate.voters.length,
                stake_total: candidate.total
              });
            }
          );
          candidates = candidates.sort((a, b) => {
            return new BN(a.stake_total.toString(), 10).lt(
              new BN(b.stake_total.toString(), 10)
            )
              ? 1
              : -1;
          });

          candidates = candidates.map((candidate, rank) => {
            return {
              rank: rank + 1,
              elected: rank < this.validatorCount ? true : false,
              ...candidate
            };
          });

          this.phragmen = {
            timestamp: data.phragmen[0].timestamp,
            block_height: data.phragmen[0].block_height,
            candidates
          };
          this.totalRows = this.phragmen.candidates.length;
        }
      },
      block: {
        query: gql`
          subscription block {
            block(limit: 1, order_by: { block_number: desc }) {
              validator_count
            }
          }
        `,
        result({ data }) {
          this.validatorCount = data.block[0].validator_count;
        }
      }
    }
  },
  head() {
    return {
      title: this.$t("pages.election-prediction.head_title", {
        networkName: network.name
      }),
      meta: [
        {
          hid: "description",
          name: "description",
          content: this.$t("pages.election-prediction.head_content", {
            networkName: network.name
          })
        }
      ]
    };
  }
};
</script>
<style>
.page-phragmen .clipboard {
  display: inline-block;
}
.page-phragmen .favorite {
  position: absolute;
  z-index: 10;
  font-size: 1.1rem;
}
.rank {
  font-size: 1.6rem;
  color: #7d7378;
}
.account {
  color: #670d35;
}
.bonded {
  color: #d75ea1;
  font-weight: 700;
  font-size: 1.3rem;
}
[data-toggle="collapse"] .fas:before {
  content: "\f078";
}
[data-toggle="collapse"][aria-expanded="false"] .fas:before {
  content: "\f054";
}
.candidates {
  color: #670d35;
}
.candidate {
  font-size: 0.9rem;
}
.candidate .value {
  color: #d75ea1;
  font-weight: 700;
}
.fee,
.unstake {
  color: #d75ea1;
  font-weight: 700;
}
.block {
  color: #d75ea1;
  font-weight: bold;
}
.block:hover {
  color: #d75ea1;
}
.tab-content .validator:nth-child(1) {
  border-top: 0;
}
.fas.fa-copy {
  cursor: pointer;
  color: #d75ea1;
  font-size: 0.9rem;
  margin-left: 0.1rem;
}
.candidate .col-md-9 .identicon {
  display: inline;
  margin-right: 0.2rem;
  cursor: copy;
}
.candidate .col-md-9 .identicon div {
  display: inline;
}
.identity {
  max-width: 80px;
}
.btn-group {
  margin-bottom: 1rem;
  display: inline-flex;
}
.btn-secondary {
  font-size: 0.8rem;
}
.page-phragmen .circle {
  display: inline-block;
  padding: 0.1rem;
  margin-left: 0.5rem;
  width: 1.4rem;
  height: 1.4rem;
  border-radius: 50%;
  text-align: center;
  font-size: 0.8rem;
  box-shadow: rgba(0, 0, 0, 0.2) 0px 3px 3px;
}
.page-phragmen .blue {
  color: white;
  background: #4682b4;
  padding-left: 0.35rem;
  padding-top: 0.1rem;
}
</style>
