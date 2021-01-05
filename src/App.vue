<template>
  <div style="width:1000px;height:800px;">
    <panel-truck screenplaySrc="https://gist.githubusercontent.com/garrettdashnelson/73b32f40cccbe30239838dd3bd63988a/raw/7d4f7efc52ee0bb0929110a4438c37d8c98f0aa5/test-screenplay.json"></panel-truck>
  </div>
</template>

<script>

import PanelTruck from './PanelTruck';

export default {
  name: 'App',
  components: {
    'panel-truck': PanelTruck
  }
}
</script>
