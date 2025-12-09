<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

let boardAPI;

function handleBoardCreated(api) {
  boardAPI = api;
}
</script>

<template>
    <TheChessboard 
      @board-created="handleBoardCreated"
    />
</template>

<style scoped>
</style>
