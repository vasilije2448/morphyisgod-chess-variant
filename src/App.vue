<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

let boardAPI;

function handleBoardCreated(api) {
  boardAPI = api;
}
</script>

<template>
  <div>
    <TheChessboard 
      @board-created="handleBoardCreated"
    />
  </div>
</template>

<style scoped>
</style>
