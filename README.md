import React, { useState } from "react";
import { View, Text, TextInput, Button, StyleSheet, Alert, SafeAreaView } from "react-native";
import axios from "axios";

export default function App() {
  const [phone, setPhone] = useState("");
  const [amount, setAmount] = useState("");

  const handlePayment = async () => {
    if (!phone || !amount) return Alert.alert("Missing", "Enter phone and amount");

    try {
      const res = await axios.post("http://your-server-ip:5000/pay", {
        phone,
        amount,
      });

      if (res.data.success) {
        Alert.alert("Prompt Sent", "Check your phone to approve payment.");
      } else {
        Alert.alert("Failed", "Payment request failed.");
      }
    } catch (error) {
      console.log(error);
      Alert.alert("Error", "Server error or invalid request.");
    }
  };

  return (
    <SafeAreaView style={styles.container}>
      <Text style={styles.title}>MoMo Payment Prompt</Text>
      <TextInput
        style={styles.input}
        placeholder="Phone (e.g. 23355XXXXXXX)"
        keyboardType="phone-pad"
        onChangeText={setPhone}
        value={phone}
      />
      <TextInput
        style={styles.input}
        placeholder="Amount (GHS)"
        keyboardType="numeric"
        onChangeText={setAmount}
        value={amount}
      />
      <Button title="Send Payment Prompt" onPress={handlePayment} />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 30,
    justifyContent: "center",
    backgroundColor: "#f9f9f9",
  },
  title: {
    fontSize: 22,
    marginBottom: 30,
    fontWeight: "bold",
    textAlign: "center",
  },
  input: {
    height: 50,
    borderWidth: 1,
    borderColor: "#ccc",
    marginBottom: 20,
    paddingHorizontal: 15,
    borderRadius: 8,
  },
});
const express = require("express");
const cors = require("cors");
const { requestToPay } = require("./momo");
require("dotenv").config();

const app = express();
app.use(cors());
app.use(express.json());

app.post("/pay", async (req, res) => {
  const { phone, amount } = req.body;
  try {
    const referenceId = await requestToPay(phone, amount);
    res.json({ success: true, referenceId });
  } catch (err) {
    console.error(err.response?.data || err.message);
    res.status(500).json({ success: false, error: err.message });
  }
});

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});
const axios = require("axios");
const { v4: uuidv4 } = require("uuid");
require("dotenv").config();

async function getAccessToken() {
  const url = `${process.env.MOMO_BASE_URL}/collection/token/`;

  const response = await axios.post(
    url,
    {},
    {
      headers: {
        "Ocp-Apim-Subscription-Key": process.env.MOMO_SUBSCRIPTION_KEY,
        Authorization:
          "Basic " +
          Buffer.from(`${process.env.MOMO_API_USER_ID}:${process.env.MOMO_API_KEY}`).toString("base64"),
      },
    }
  );

  return response.data.access_token;
}

async function requestToPay(phone, amount) {
  const referenceId = uuidv4();
  const token = await getAccessToken();

  const url = `${process.env.MOMO_BASE_URL}/collection/v1_0/requesttopay`;

  await axios.post(
    url,
    {
      amount: amount.toString(),
      currency: "GHS",
      externalId: referenceId,
      payer: {
        partyIdType: "MSISDN",
        partyId: phone,
      },
      payerMessage: "Payment prompt",
      payeeNote: "Thanks for payment",
    },
    {
      headers: {
        Authorization: `Bearer ${token}`,
        "X-Reference-Id": referenceId,
        "X-Target-Environment": process.env.MOMO_TARGET_ENV,
        "Ocp-Apim-Subscription-Key": process.env.MOMO_SUBSCRIPTION_KEY,
        "Content-Type": "application/json",
      },
    }
  );

  return referenceId;
}

module.exports = { requestToPay };
