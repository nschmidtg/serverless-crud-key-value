<template>
  <div>
    <template v-for="item in items">
      <item
        :item="item"
        :key="item.key"
        @delete-item="checkCredentialsBeforeOperation(deleteItem,$event)"
      />
    </template>
    <input
      v-model="key"
      type="text"
      placeholder="key"
    >
    <input
      v-model="value"
      type="text"
      placeholder="value"
    >
    <button
      @click="checkCredentialsBeforeOperation(addItem)"
    >
      Add item to table
    </button>
  </div>
</template>

<script>
import Item from './components/Item'
const AWS = require('aws-sdk')
// Set the region where your identity pool exists
AWS.config.region = 'REGION_OF_YOUR_IDENTITY_POOL'
// Configure the credentials provider to use your identity pool
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: 'YOUR_AWS_COGNITO_IDENTITY_POOL_ID'
})
export default {
  components: {
    item: Item
  },
  data () {
    return {
      credentials: null,
      key: '',
      value: '',
      items: [],
      tableName: 'YOUR_DYNAMODB_TABLE_NAME',
      apiVersion: '2012-08-10',
      docClient: null
    }
  },
  created () {
    console.log(process.env.VUE_APP_IDENTITY)
    this.docClient = new AWS.DynamoDB.DocumentClient({
      apiVersion: this.apiVersion
    })
    this.checkCredentialsBeforeOperation(this.getItems)
  },
  methods: {
    getCredentialsPromise () {
      let that = this
      console.log('in getCredentials()')
      var promise = AWS.config.credentials.getPromise()
      promise.then(
        function () {
          console.log('result...')
          var creds = {
            accessKeyId: AWS.config.credentials.accessKeyId,
            secretAccessKey: AWS.config.credentials.secretAccessKey,
            sessionToken: AWS.config.credentials.sessionToken
          }
          console.log('got credentials: ' + JSON.stringify(creds, null, 2))
          that.credentials = creds
        },
        function (err) {
          console.log('err...' + JSON.stringify(err, null, 2))
          that.credentials = null
        }
      )
      return promise
    },
    checkCredentialsBeforeOperation (authenticatedOperation, paramsOfOperation = null) {
      console.log('in checkCredentialsBeforeOperation')
      if (this.credentials === null) {
        console.log('in checkCredentialsBeforeOperation, no credentials... getting credentials')
        this.getCredentialsPromise()
          .then((exito) => {
            console.log(this.credentials)
            authenticatedOperation(paramsOfOperation)
          })
      } else {
        console.log('in checkCredentialsBeforeOperation, with credentials')
        authenticatedOperation(paramsOfOperation)
      }
    },
    getItems () {
      console.log('in getItems()')
      var params = {
        TableName: this.tableName
      }
      this.docClient.scan(params, (err, data) => {
        if (err) {
          console.log(
            'error getting item, err: ' + JSON.stringify(err, null, 2)
          )
        } else {
          this.items = data.Items
        }
      })
    },
    addItem () {
      console.log('in addItem()')
      // adds given item
      let item = {
        key: this.key,
        value: this.value
      }
      var params = {
        TableName: this.tableName,
        Item: item
      }
      console.log('in addItem(), have credentials, adding item...')
      this.docClient.put(params, (err, data) => {
        if (err) {
          console.log('Error adding item', JSON.stringify(err, null, 2))
        } else {
          console.log('Success adding item', JSON.stringify(data, null, 2))
          console.log(data)
          this.items.push(item)
          this.key = ''
          this.value = ''
        }
      })
    },
    deleteItem (item) {
      console.log('in deleteItem()')
      var params = {
        TableName: this.tableName,
        Key: {
          key: item.key
        },
        ReturnValues: 'NONE'
      }
      console.log('in deleteItem(), got credentials....')
      this.docClient.delete(params, (err, data) => {
        if (err) {
          console.log(
            'error deleting item, err: ' + JSON.stringify(err, null, 2)
          )
        } else {
          console.log(
            'success deleting item: ' + JSON.stringify(data, null, 2)
          )
          this.items.splice(this.items.indexOf(item), 1)
        }
      })
    }
  }
}
</script>
