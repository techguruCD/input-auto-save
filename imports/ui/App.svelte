<script>
  import AutoInput from './AutoInput.svelte'
  import { Meteor } from "meteor/meteor";
  import { LinksCollection } from '../api/links';
  
  let counter = 0;
  const addToCounter = () => {
    counter += 1;
  }
  
  let subIsReady = false;
  $m: {
    const handle = Meteor.subscribe("links.all");
    subIsReady = handle.ready();
  }

  // more information about $m at https://atmospherejs.com/zodern/melte#tracker-statements
  let links;
  $m: links = LinksCollection.find().fetch();
</script>


<div class="container mx-auto h-screen p-8">
  <AutoInput unique_id="1" />
</div>
