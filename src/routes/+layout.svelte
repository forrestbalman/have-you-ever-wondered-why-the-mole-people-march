<script>
    import "bootstrap/dist/css/bootstrap.min.css";
    import "bootstrap-icons/font/bootstrap-icons.css";
    import { onMount } from "svelte";

    onMount(() => {
        import("bootstrap/dist/js/bootstrap.min.js");
    });
</script>

<slot></slot>
