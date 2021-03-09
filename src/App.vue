<template>
  <div id="app">
    <amplify-authenticator>
      <amplify-sign-in
        slot="sign-in"
        hide-sign-up="true"
        header-text="Connectez-vous (placeholder logs)"
        submit-button-text="Connexion"
        :form-fields.prop="signInFields"
      ></amplify-sign-in>
      <!-- <amplify-sign-in slot="sign-in" header-text="Connectez-vous avec le compte principal"  submit-button-text="Connexion"></amplify-sign-in> -->
      <div class="welcome">
        <amplify-sign-out button-text="Déconnexion"></amplify-sign-out>
        <!-- <h1>Bienvenue {{ user.username }}</h1> -->
        <h1>Bienvenue sur la boîte à idées ! Partagez vos idées</h1>
      </div>
      <div class="form-body">
        <form v-on:submit.prevent autocomplete="off">
          <h2>Proposer une idée d'application</h2>
          <div>
            <label for="name">Nom de l'application: </label>
            <input
              class="input"
              id="name"
              v-model="form.name"
              autocomplete="off"
            />
          </div>
          <div>
            <label for="description">Description de l'application: </label>
            <textarea
              class="input"
              id="description"
              v-model="form.description"
              autocomplete="off"
              rows="4"
              style="display: block; width: 100%"
            />
          </div>
          <div>
            <label for="plateforme">Plateforme: </label>
            <select id="plateforme" v-model="form.plateforme" name="plateforme">
              <option value="web">Web</option>
              <option value="mobile">Mobile</option>
              <option value="all">Toutes</option>
            </select>
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
            <h3 style="margin: 0 0 1rem 0; padding 0">
              <strong>{{ restaurant.name }}</strong>
            </h3>
            <div class="description">{{ restaurant.description }}</div>
            <div style="font-weight: 600; padding-top: 1rem" class="plateforme">
              <i>{{ restaurant.plateforme }}</i>
            </div>
            <div class="remove">
              <button class="button" @click="delete_restaurant(restaurant)">
                Supprimer
              </button>
            </div>
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
      signInFields: [
        {
          type: "email",
          label: "Utilisateur",
          placeholder: "admin",
          required: true,
        },
        {
          type: "password",
          label: "Mot de passe",
          placeholder: "admin_password",
          required: true,
        },
      ],
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
      // set current user and load data after login
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
        console.log("Erreur de chargement des idées", error);
      } finally {
        this.loading = false;
      }
    },
    async create_restaurant() {
      const { name, description, plateforme } = this.form;
      if (!name || !description || !plateforme) return;

      const restaurant = {
        name,
        description,
        plateforme,
        clientId: this.clientId,
      };

      try {
        const response = await API.graphql(
          graphqlOperation(createRestaurant, { input: restaurant })
        );
        console.log("idée créé !");
        this.restaurants = [
          ...this.restaurants,
          response.data.createRestaurant,
        ];
        this.form = { name: "", description: "", plateforme: "" };
      } catch (error) {
        console.log("Erreur à la création d'idée...", error);
      }
    },
    async delete_restaurant(restaurant) {
      if (restaurant) {
        try {
          const { id } = restaurant;
          await API.graphql(
            graphqlOperation(deleteRestaurant, { input: { id: id } })
          );
          console.log("idée supprimé");
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
body {
  margin: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

.card-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
}

.card {
  margin: 2rem 1rem;
  width: 300px;
  border: 1px solid black;
  padding: 0.8rem;
  box-shadow: 1px 1px 1px black;
  border-radius: 5px;
}

.description,
.name,
.plateforme {
  margin: 0.2rem 0;
}

.form-body {
  width: 50%;
  margin: 0 auto;
  padding: 1rem;
  border: 1px solid black;
  border-radius: 5px;
  box-shadow: 1px 1px 1px black;
}

.description {
  word-wrap: break-word;
  text-align: left;
}

.form-body form div {
  text-align: left;
  padding: 0.5rem 0;
}

h2 {
  margin: 0 0 1rem 0;
  text-align: left;
}

button {
  border: 1px solid black;
  padding: 0.5rem 1rem;
  margin: 0.2rem;
  box-shadow: 1px 1px 1px black;
  background-color: #ff6347;
  margin-top: 1rem;
}

button:first-of-type {
  background-color: #209cee;
}
button:nth-of-type(2) {
  background-color: #92cc41;
}

:root {
  --amplify-primary-color: #ff6347;
  --amplify-primary-tint: #ff7359;
  --amplify-primary-shade: #e0573e;
}
</style>
