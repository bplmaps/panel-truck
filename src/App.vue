<template>
  <div style="width:1000px;height:800px;">
    <panel-truck screenplaySrc="https://gist.githubusercontent.com/garrettdashnelson/725fa9dd3ff1987458fda4d8183236e7/raw/c36da07c2970f0a0bcbd2d58b4df96930a81098a/paneltruck-0.2-screenplay.json"></panel-truck>
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
