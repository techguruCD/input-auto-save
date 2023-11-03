<script>
  import AutoInput from "./AutoInput.svelte";
  import { Meteor } from "meteor/meteor";
  import { LinksCollection } from "../api/links";

  let alternativeItems = [
    "Alternative Item 1",
    "Alternative Item 2",
    "Alternative Item 3",
    "Alternative Item 4",
    "Alternative Item 5",
    "Alternative Item 6",
    "Alternative Item 7",
    "Alternative Item 8",
    "Alternative Item 9",
    "Alternative Item 10",
    "Alternative Item 11",
    "Alternative Item 12",
    "Alternative Item 13",
    "Alternative Item 14",
    "Alternative Item 15",
    "Alternative Item 16",
    "Alternative Item 17",
    "Alternative Item 18",
    "Alternative Item 19",
  ];

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
  <AutoInput unique_id="1" alternativeItems={alternativeItems}/>
</div>
