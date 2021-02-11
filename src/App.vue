<template>
  <div id="app">
    <div class="app-header">
      <div class="app-logo">
        <img
          src="https://aws-amplify.github.io/images/Logos/Amplify-Logo-White.svg"
          alt="AWS Amplify"
        />
      </div>
      <h1>Welcome to my AAAAAAAAAAAA</h1>
    </div>
    <amplify-authenticator>
      <div class="welcome">
        <h1>Hey, {{ user.username }}</h1>
        <amplify-sign-out></amplify-sign-out>
      </div>
      <div class="form-body">
        <form v-on:submit.prevent autocomplete="off">
          <div>
            <label for="name">Name:</label>
            <input
              class="input"
              id="name"
              v-model="form.name"
              autocomplete="off"
            />
          </div>
          <div>
            <label for="description"></label>
            <input
              class="input"
              id="description"
              v-model="form.description"
              autocomplete="off"
            />
          </div>
          <div>
            <label for="city"></label>
            <input
              class="input"
              id="city"
              v-model="form.city"
              autocomplete="off"
            />
          </div>
          <button @click="create_restaurant">Créér</button>
          <button @click="get_data">Rafraîchir</button>
        </form>
      </div>
      <div class="app-body">
        <div class="loading" v-if="loading">Chargement...</div>
        <div class="card-container" v-if="!loading">
          <div
            class="card"
            :key="restaurant.id"
            v-for="restaurant of sorted_restaurants"
          >
            <div class="remove">
              <button class="button" @click="delete_restaurant(restaurant)">
                Supprimer
              </button>
            </div>
            <div class="name">{{ restaurant.city }}</div>
            <div class="price">{{ restaurant.name }}</div>
            <div class="symbol">{{ restaurant.description }}</div>
          </div>
        </div>
      </div>
    </amplify-authenticator>
  </div>
</template>

<script>
import { AuthState, onAuthUIStateChange } from "@aws-amplify/ui-components";
import { API, graphqlOperation } from "aws-amplify";
import { listRestaurants } from "./graphql/queries";
import { createRestaurant, deleteRestaurant } from "./graphql/mutations";
import {
  onCreateRestaurant,
  onDeleteRestaurant,
} from "./graphql/subscriptions";

export default {
  name: "App",
  data() {
    return {
      user: {},
      restaurants: [],
      form: {},
      loading: true,
    };
  },
  computed: {
    sorted_restaurants() {
      let restaurants = [...this.restaurants];
      return restaurants.sort((a, b) => a.name.localeCompare(b.name));
    },
  },
  created() {
    // auth state management
    onAuthUIStateChange((state, user) => {
      // set current user and laod data after login
      if (state === AuthState.SignedIn) {
        this.user = user;
        this.get_data();
      }
    });
    //subscribe to changes
    API.graphql(graphqlOperation(onCreateRestaurant)).subscribe(
      (source_data) => {
        const new_restaurant = source_data.value.data.onCreateRestaurant;
        if (new_restaurant) {
          //skip our own mutations and duplicates
          if (this.restaurants.some((r) => r.id == new_restaurant.id)) return;

          this.restaurants = [new_restaurant, ...this.restaurants];
        }
      }
    );

    API.graphql(graphqlOperation(onDeleteRestaurant)).subscribe(
      (source_data) => {
        const deleted_restaurant = source_data.value.data.onDeleteRestaurant;
        if (deleted_restaurant) {
          this.restaurants = this.restaurants.filter(
            (r) => r.id != deleted_restaurant.id
          );
        }
      }
    );
  },
  methods: {
    async get_data() {
      try {
        this.laoding = true;
        const response = await API.graphql(graphqlOperation(listRestaurants));
        this.restaurants = response.data.listRestaurants.items;
      } catch (error) {
        console.log("Erreur de chargement des restaurants", error);
      } finally {
        this.loading = false;
      }
    },
    async create_restaurant() {
      const { name, description, city } = this.form;
      if (!name || !description || !city) return;

      const restaurant = { name, description, city, clientId: this.clientId };

      try {
        const response = await API.graphql(
          graphqlOperation(createRestaurant, { input: restaurant })
        );
        console.log("Restaurant créé !");
        this.restaurants = [
          ...this.restaurants,
          response.data.createRestaurant,
        ];
        this.form = { name: "", description: "", city: "" };
      } catch (error) {
        console.log("Erreur à la création de restaurant...", error);
      }
    },
    async delete_restaurant(restaurant) {
      if (restaurant) {
        try {
          const { id } = restaurant;
          await API.graphql(
            graphqlOperation(deleteRestaurant, { input: { id: id } })
          );
          console.log("Restaurant supprimé");
          this.restaurants = this.restaurants.filter(
            (r) => r.id != restaurant.id
          );
        } catch (error) {
          console.log("Erreur à la suppression...", error);
        }
      }
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
