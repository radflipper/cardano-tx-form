<template>
  <v-container>
    <v-row class="text-center">
      <v-col cols="12">
        <v-btn v-if="account" color="primary" @click="selectWallet">
          <v-icon>mdi-check</v-icon> Connected
        </v-btn>
        <v-btn v-else color="primary" @click="selectWallet"> Connect </v-btn>

        <v-dialog v-model="connectDialog">
          <v-card>
            <v-card-text> Choose wallet to connect: </v-card-text>
            <v-card-actions>
              <v-btn
                color="primary"
                :wallet="wallet"
                block
                @click="connectWallet(wallet.nami)"
              >
                {{ wallet.nami }}
              </v-btn>
            </v-card-actions>
            <v-card-actions>
              <v-btn
                color="primary"
                :wallet="wallet"
                block
                @click="connectWallet(wallet.ccvault)"
              >
                {{ wallet.ccvault }}
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
        <v-form>
          <v-text-field v-model="sendAmount" label="Amount"></v-text-field>
          <v-text-field
            v-model="sendAddress"
            label="Send to Address"
          ></v-text-field>
          <v-btn @click="submit">Submit TX</v-btn>
        </v-form>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import * as Cardano from "@emurgo/cardano-serialization-lib-asmjs";
import axios from "axios";
import { Buffer } from "buffer";

export default {
  name: "App",
  data: () => ({
    account: {},
    selectedWallet: "",
    connectDialog: false,
    sendDialog: false,
    wallet: {
      nami: "nami",
      ccvault: "ccvault",
    },
    sendAmount: "",
    sendAddress: "",
  }),
  methods: {
    selectWallet() {
      this.connectDialog = true;
    },
    async connectWallet(selectedWallet) {
      this.account = await window.cardano[selectedWallet].enable();
      this.connectDialog = false;
    },
    async isConnected() {
      if (typeof window.cardano != undefined) {
        const nami = await window.cardano.nami.isEnabled();
        const ccvault = await window.cardano.ccvault.isEnabled();
        const selection = nami ? "nami" : ccvault ? "ccvault" : false;
        selection
          ? (this.account = await window.cardano[selection].enable())
          : (this.account = false);
        console.log(this.account);
      }
    },
    async getParameters() {
      const id = process.env.VUE_APP_BLOCKFROST_ID;

      const result = await axios.get(
        "https://cardano-testnet.blockfrost.io/api/v0/epochs/latest/parameters",
        {
          headers: {
            project_id: id,
          },
        }
      );

      return {
        linearFee: {
          minFeeA: result.data.min_fee_a.toString(),
          minFeeB: result.data.min_fee_b.toString(),
        },
        poolDeposit: result.data.pool_deposit,
        keyDeposit: result.data.key_deposit,
        coinsPerUtxoWord: result.data.coins_per_utxo_word,
        maxValSize: result.data.max_val_size,
        priceMem: result.data.price_mem,
        priceStep: result.data.price_step,
        maxTxSize: parseInt(result.data.max_tx_size),
      };
    },

    async transaction(sendAmount, paymentAddress, account) {
      const address = await account.getChangeAddress();
      const changeAddress = Cardano.Address.from_bytes(
        Buffer.from(address, "hex")
      ).to_bech32();
      const protocolParameters = await this.getParameters();

      const txConfig = Cardano.TransactionBuilderConfigBuilder.new()
        .coins_per_utxo_word(
          Cardano.BigNum.from_str(protocolParameters.coinsPerUtxoWord)
        )
        .fee_algo(
          Cardano.LinearFee.new(
            Cardano.BigNum.from_str(protocolParameters.linearFee.minFeeA),
            Cardano.BigNum.from_str(protocolParameters.linearFee.minFeeB)
          )
        )
        .key_deposit(Cardano.BigNum.from_str(protocolParameters.keyDeposit))
        .pool_deposit(Cardano.BigNum.from_str(protocolParameters.poolDeposit))
        .max_tx_size(protocolParameters.maxTxSize)
        .max_value_size(protocolParameters.maxValSize)
        .prefer_pure_change(true)
        .build();

      const txBuilder = Cardano.TransactionBuilder.new(txConfig);

      txBuilder.add_output(
        Cardano.TransactionOutputBuilder.new()
          .with_address(Cardano.Address.from_bech32(paymentAddress))
          .next()
          .with_value(Cardano.Value.new(Cardano.BigNum.from_str(sendAmount)))
          .build()
      );

      const utxos = Cardano.TransactionUnspentOutputs.new();
      const plainUtxos = (await account.getUtxos()).map((utxo) =>
        Cardano.TransactionUnspentOutput.from_bytes(Buffer.from(utxo, "hex"))
      );
      plainUtxos.map((currentUtxo) => {
        utxos.add(currentUtxo);
      });

      txBuilder.add_inputs_from(utxos, 0);
      console.log(txBuilder.get_total_input);
      txBuilder.add_change_if_needed(
        Cardano.Address.from_bech32(changeAddress)
      );

      const txBody = txBuilder.build();
      const transaction = Cardano.Transaction.new(
        txBuilder.build(),
        Cardano.TransactionWitnessSet.new()
      );
      const witness = await account.signTx(
        Buffer.from(transaction.to_bytes(), "hex").toString("hex")
      );

      const signedTx = Cardano.Transaction.new(
        txBody,
        Cardano.TransactionWitnessSet.from_bytes(Buffer.from(witness, "hex")),
        undefined // transaction metadata
      );

      const txHash = await account.submitTx(
        Buffer.from(signedTx.to_bytes()).toString("hex")
      );
      return txHash;
    },
    async submit() {
      this.sendDialog = false;
      await this.transaction(
        this.sendAmount.replace(".", "") + "000000",
        this.sendAddress,
        this.account
      );
    },
    async getAddress() {
      this.sendDialog = true;
      await this.$recaptchaLoaded();
      // Execute reCAPTCHA with action "login".
      const token = await this.$recaptcha("login");

      // Do stuff with the received token.
      axios.post(this.apiUrl + "getNfts", { token: token }).then((res) => {
        const amount = res.data.adaToSend;
        this.sendAmount = amount;

        this.sendAddress = res.data.paymentAddress;
      });
    },
  },

  mounted() {
    this.isConnected();
  },
};
</script>
