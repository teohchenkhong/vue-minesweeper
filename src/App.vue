
<template>
  <header>
    <nav class="bg-blue-500 p-4">
      <div class="container mx-auto flex justify-between items-center">
        <!-- <div>
          <a href="#" class="text-white text-lg font-semibold">Your Logo</a>
        </div> -->
        <div class="space-x-4">
          <RouterLink 
            v-for = "({path, name}) in routes"
            :class = "'text-white ' + ($route.path === path ? 'border-b border-white' : '')" 
            :to="path"
          >
            {{name}}
          </RouterLink>
        </div>
      </div>
    </nav>
  </header>

  <RouterView />
</template>

<script setup>
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'

const routes= [
  {
    path: "/minesweeper"
    ,name: "Minesweeper"
  }
  ,{
    path: "/othello"
    ,name: "Othello"
  }
  ,{
    path: "/chinese-chess"
    ,name: "Chinese Chess"
  }
  ,{
    path: "/tetris"
    ,name: "Tetris"
  }
];
</script>


