<!-- Renders search field and search results -->

<template>
  <div class="card">
    <div class="card-header">
      <p class="card-header-title">WISH LIST</p>
    </div>
    <div class="card-content">
      <!-- Search field begins -->
      <b-field>
        <b-input
          v-model="keyword"
          @input="searchIconClick"
          type="search"
          placeholder="Search..."
          icon="magnify"
          icon-clickable
          @icon-click="searchIconClick"
        >
        </b-input>
      </b-field>
      <!-- Search field ends -->
      <!-- Search results begins -->
      <div v-if="loaded">
        <article
          class="media"
          v-for="plant in results"
          v-if="plant.common_name"
          :key="plant.id"
        >
          <figure class="media-left">
            <thumbnail :plant="plant"></thumbnail>
          </figure>
          <div class="media-content">
            <div class="content">
              <p>
                <strong>{{ plant.common_name }}</strong>
                <br />
                {{ plant.family_common_name }}
              </p>
            </div>
            <nav :class="mobileStyle()">
              <div class="level-left">
                <b-button
                  size="is-small"
                  type="is-info"
                  icon-left="plus-circle"
                  @click="wishButtonClick(plant.id, plant.slug)"
                  :active="wishClicked.includes(plant.id)"
                >
                  <span v-if="!wishClicked.includes(plant.id)">
                    Add to Wishlist
                  </span>
                  <span v-else>Included in Wishlist</span>
                </b-button>
                <b-button
                  v-if="gardenId > -1"
                  type="is-success"
                  size="is-small"
                  icon-left="plus-circle"
                  @click="gardenButtonClick(plant.id, plant.slug)"
                >
                  Add to Garden
                </b-button>
              </div>
            </nav>
          </div>
        </article>
      </div>
      <div v-else>
        <wishListSkeleton></wishListSkeleton>
      </div>
      <!-- Search results ends -->
    </div>
  </div>
</template>

<script>
import PlantThumbnail from "./PlantThumbnail.vue";
import WishListSkeleton from "./WishListSkeleton.vue";
import { isMobile } from "mobile-device-detect";
import axios from "axios";
import debounce from "lodash";

export default {
  name: "Wish",
  components: {
    wishListSkeleton: WishListSkeleton,
    thumbnail: PlantThumbnail,
  },
  data() {
    return {
      loaded: false,
      keyword: "",
      results: [],
      wishClicked: [],
      gardenId: this.$route.params.gardenId || -1,
      reloader: 0,
      mobile: isMobile,
    };
  },
  created() {
    this.searchIconClick = _.debounce(this.searchIconClick, 3000);
  },
  props: ["plant", "user"],

  methods: {
    mobileStyle() {
      let style = "level ";
      style += this.mobile ? "is-mobile" : "";
      return style;
    },
    emptyResultsToast(isSearch) {
      let text;
      if (!isSearch) {
        // Message for empty results on mount
        text = `No plants found in your wishlist.`;
      } else {
        // Message for empty results on search
        text = `No plants found during search.`;
      }
      this.$buefy.toast.open({
        duration: 5000,
        message: text,
        type: "is-warning",
      });
    },
    searchIconClick() {
      this.loaded = false;
      axios
        .get("/api/search", {
          params: {
            q: this.keyword,
          },
        })
        .then((res) => {
          if (res.data.length === 0) {
            this.emptyResultsToast(true);
          } else {
            this.results = res.data.map((plant) => {
              plant.photo_url = plant.image_url;
              delete plant.image_url;
              return plant;
            });
          }
        })
        .catch((err) => {
          console.error(err);
        });
      this.keyword = "";
      this.loaded = true;
    },
    clearIconClick() {
      this.search = "";
    },
    wishButtonClick(treflePlantId, treflePlantSlug) {
      const wishIndex = this.wishClicked.indexOf(treflePlantId);
      if (wishIndex > -1) {
        this.wishClicked.splice(wishIndex, 1);
        axios({
          method: "DELETE",
          url: "/wishlist",
          params: {
            userId: this.user.id,
            plantId: treflePlantId,
          },
        })
          .then((res) => {
            this.$log.info(res);
          })
          .catch((err) => {
            console.error(err);
          });
      } else {
        this.wishClicked.push(treflePlantId);
        axios
          .post("/wishlist", {
            userId: this.user.id,
            plantId: treflePlantId,
            slug: treflePlantSlug,
          })
          .then((res) => {
            this.$log.info(res);
          })
          .catch((err) => {
            console.error(err);
          });
      }
      this.keyword = "";
    },
    gardenButtonClick(treflePlantId, treflePlantSlug) {
      this.gardenId = this.$route.params.gardenId;
      axios
        .post("/garden/addplant", {
          gardenId: this.$route.params.gardenId,
          plantId: treflePlantId,
          slug: treflePlantSlug,
        })
        .then((res) => {
          this.$log.info(res);
          this.$buefy.toast.open({
            type: "is-success",
            duration: 1000,
            message: `Added ${res.data.plantName}`,
          });
          // this.keyword = "";
          // this.$router.push({
          //   path: "/garden",
          //   query: { id: this.gardenId },
          // });
        })
        .catch((err) => {
          console.error(err);
        });
    },
  },
  mounted() {
    this.loaded = false;

    if (this.plant !== undefined) {
      this.keyword = this.plant;
      this.searchIconClick();
    }
    axios({
      method: "GET",
      url: "/wishlist",
      params: { userId: this.user.id },
    })
      .then(({ data }) => {
        if (data.length === 0) {
          this.emptyResultsToast(false);
        } else {
          this.results = data
            .filter((plant) => {
              if (!this.wishClicked.includes(plant.plant.id_trefle)) {
                this.wishClicked.push(plant.plant.id_trefle);
                return plant;
              }
            })
            .map((uniquePlant) => {
              return {
                id: uniquePlant.plant.id_trefle,
                common_name: uniquePlant.plant.common_name,
                slug: uniquePlant.plant.slug,
                photo_url: uniquePlant.plant.photo_url,
              };
            });
        }
      })
      .catch((err) => {
        console.error(err);
      });
    this.loaded = true;
  },
  beforeRouteUpdate(to, from, next) {
    this.keyword = to.query.name;
    this.gardenId = to.query.gardenId;
    this.searchIconClick();
    next();
  },
};
</script>

<style lang="sass">
.level.is-mobile .level-left
  align-items: start
  flex-direction: column
</style>
