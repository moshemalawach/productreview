<template>
  <div id="app">
    <notifications group="wallet" />
    <b-modal id="writeModal" ref="writeModal" hide-footer title="Write a review" v-model="writeShow">
      <b-form-group
      id="fieldset1"
      label="Product ID"
      label-for="form_product_id"
      >
        <b-form-input id="form_product_id" v-model.trim="form_product_id"></b-form-input>
      </b-form-group>
      <b-form-group
      id="fieldset1"
      label="Your comment"
      label-for="form_comment"
      >
        <b-form-textarea id="form_comment" v-model="form_comment"></b-form-textarea>
      </b-form-group>

      <button class="btn btn-lg btn-block btn-primary mb-3" v-if="form_product_id&&form_comment" v-on:click="broadcast">
        Submit
      </button>
    </b-modal>
    <b-container>
      <div class="d-flex justify-content-between">
        <h1>NULS Example smart contract dApp</h1>
        <div>
          <b-link v-if="!account" @click="login">
            Log-In
          </b-link>
          <p class="badge" v-else>
            Welcome {{account.name}}!
            <b-link @click="logout">
              Log-Out
            </b-link>
          </p>
          <b-link v-b-tooltip.hover title="Refresh" @click="refresh">🔄</b-link>
        </div>
      </div>
      <b-row class="mt-4">
        <b-col sm="12" md="4" lg="2">
          <h4>Products</h4>
          <b-list-group>
            <b-list-group-item v-for="product of products">
              <b-link @click="product_id=product">
                {{product}}
              </b-link>
            </b-list-group-item>
          </b-list-group>
        </b-col>
        <b-col sm="12" md="8" lg="10">
          <div class="d-flex justify-content-between">
            <h2 v-if="product_id">Reviews of {{product_id}}</h2>
            <b-button v-if="account" @click="write_review(product_id)" class="mb-4">
              Write a review
            </b-button>
          </div>
          <b-card v-for="review of reviews" class="mb-2"
            :title="`Review from ${review.writer}`">
            {{review.comments}}
          </b-card>
          <div class="d-flex flex-row-reverse">
            <b-button v-if="account&&product_id" @click="write_review(product_id)" class="mt-4">
              Write a review
            </b-button>
          </div>
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script>
import {call_view_method,
        prepare_contract_call_tx} from 'nulsworldjs/src/api/contracts'
import {private_key_to_public_key,
  address_from_hash,
  public_key_to_hash
} from 'nulsworldjs/src/model/data.js'
import {broadcast} from 'nulsworldjs/src/api/create'
import {Transaction} from 'nulsworldjs/src/model/transaction'

var api_server = 'https://testnet.nuls.world'
var network_id = 261
var contract_address = 'TTb1oshGEhmiM513bhJ2CLAwTJ7HU6hs'

var hexRegEx = /([0-9]|[a-f])/gim

function isHex (input) {
  return typeof input === 'string' &&
    (input.match(hexRegEx) || []).length === input.length
}

export default {
  name: 'app',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      products: [],
      product_id: null,
      reviews: [],
      writeShow: false,
      account: null,
      chain_id: 261,
      private_key: null,
      form_product_id: '',
      form_comment: ''
    }
  },
  watch: {
    async product_id() {
      this.reviews = []
      await this.get_reviews(this.product_id)
    }
  },
  methods: {
    async refresh() {
      await this.update_products()
      if (this.product_id)
        await this.get_reviews(this.product_id)
    },
    async update_products() {
      let results = await call_view_method(
        contract_address, 'getAllProductIds', [], {
        'api_server': api_server
      })
      let result = JSON.parse(results)
      this.products = result
    },
    async get_reviews(product_id) {
      let results = await call_view_method(
        contract_address, 'getReviews', [product_id], {
        'api_server': api_server
      })
      let result = JSON.parse(results)
      this.reviews = result
    },
    check_pkey() {
      if (!isHex(this.private_key)) { return false }
      if (!this.private_key) { return false }
      if ((this.private_key.length === 66) && (this.private_key.substring(0, 2) === '00')) {
        this.private_key = this.private_key.substring(2, 66)
        return true
      }
      if (this.private_key.length !== 64) { return false }
      try {
        let prvbuffer = Buffer.from(this.private_key, 'hex')
        let pub = private_key_to_public_key(prvbuffer)
        return true
      } catch (e) {
        return false
      }
    },
    async login() {
      this.private_key = prompt("Please enter your private key:")
      if (!this.check_pkey()) {
        alert("Private key is invalid.")
        return
      }

      let prvbuffer = Buffer.from(this.private_key, 'hex')
      let pub = private_key_to_public_key(prvbuffer)
      let hash = public_key_to_hash(pub, {
        chain_id: this.chain_id
      })
      let address = address_from_hash(hash)
      // Vue.set(this, 'public_key', pub);
      let public_key = pub.toString('hex')
      let address_hash = hash.toString('hex')
      this.account = {
        'name': address,
        'private_key': this.private_key,
        'public_key': public_key,
        'address': address
      }
    },
    async logout() {
      this.account = null
    },
    write_review(product_id) {
      this.form_product_id = product_id
      this.form_comment = ''
      this.writeShow = true
    },
    async broadcast() {
      let tx = await prepare_contract_call_tx(
        this.account.address,
        contract_address,
        'writeReview', [ // Items in the arguments should be arrays, even of one item.
            [this.form_product_id],
            [this.form_comment]
          ], {
          'api_server': api_server
        })
      console.log(tx)

      tx.sign(Buffer.from(this.account.private_key, 'hex'))
      console.log(tx)
      let signed_tx = tx.serialize().toString('hex')
      console.log(signed_tx)

      let other_tx = new Transaction()
      other_tx.parse(Buffer.from(signed_tx, 'hex'))
      console.log(other_tx)

      let response = await broadcast(signed_tx, {'api_server': api_server})
      if ((response !== undefined)&&(response !== null)) {
        this.$notify({
          group: 'wallet',
          title: 'Transaction sent',
          'type': 'success',
          text: 'TX hash: ' + response
        })
      } else {
        this.$notify({
          group: 'wallet',
          title: 'Error',
          'type': 'error',
          text: 'An error occured while trying to broadcast the TX.'
        })
      }
    }
  },
  async mounted() {
    console.log('blah')
    await this.update_products()
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}

h1, h2 {
  font-weight: normal;
  text-align: center;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
