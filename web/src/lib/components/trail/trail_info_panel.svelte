<script lang="ts">
    import { goto } from "$app/navigation";
    import SummitLogCard from "$lib/components/summit_log/summit_log_card.svelte";
    import Tabs from "$lib/components/base/tabs.svelte";
    import TrailDropdown from "$lib/components/trail/trail_dropdown.svelte";
    import WaypointCard from "$lib/components/waypoint/waypoint_card.svelte";
    import type { Trail } from "$lib/models/trail";
    import { currentUser } from "$lib/stores/user_store";
    import { getFileURL } from "$lib/util/file_util";
    import {
        formatDistance,
        formatElevation,
        formatTimeHHMM,
    } from "$lib/util/format_util";
    import { createMarkerFromWaypoint } from "$lib/util/leaflet_util";
    import "$lib/vendor/leaflet-elevation/src/index.css";
    import type { Icon, Map, Marker } from "leaflet";
    import "leaflet.awesome-markers/dist/leaflet.awesome-markers.css";
    import "leaflet/dist/leaflet.css";
    import "photoswipe/style.css";
    import { onMount } from "svelte";
    import { _ } from "svelte-i18n";
    import PhotoGallery from "../photo_gallery.svelte";

    export let trail: Trail;
    export let mode: "overview" | "map" = "map";
    export let markers: Marker[] = [];

    const tabs = [
        $_("description"),
        $_("waypoints"),
        $_("photos"),
        $_("summit-book"),
    ];

    let map: Map;

    let activeTab = 0;

    let thumbnail = trail.photos.length
        ? getFileURL(trail, trail.photos[trail.thumbnail])
        : "/imgs/default_thumbnail.webp";

    let openGallery: (idx?: number) => void;

    onMount(async () => {
        if (mode == "overview") {
            const L = (await import("leaflet")).default;
            await import("leaflet-gpx");
            await import("leaflet.awesome-markers");

            map = L.map("map").setView([0, 0], 2);
            map.attributionControl.setPrefix(false);

            L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
                attribution: "© OpenStreetMap contributors",
            }).addTo(map);

            const gpxLayer = new L.GPX(trail.expand.gpx_data!, {
                async: true,
                polyline_options: {
                    className: "lightblue-theme elevation-polyline",
                    opacity: 0.75,
                    weight: 5,
                },
                gpx_options: {
                    parseElements: ["track"] as any,
                },
                marker_options: {
                    startIcon: L.AwesomeMarkers.icon({
                        icon: "circle-half-stroke",
                        prefix: "fa",
                        markerColor: "cadetblue",
                        iconColor: "white",
                    }) as Icon,
                    startIconUrl: "",
                    endIconUrl: "",
                    shadowUrl: "",
                },
            })
                .on("loaded", function (e) {
                    map.fitBounds(e.target.getBounds());
                })
                .addTo(map);

            for (const waypoint of trail.expand.waypoints) {
                const marker = createMarkerFromWaypoint(L, waypoint);
                marker.addTo(map);
                markers.push(marker);
            }
        }
    });

    function openMarkerPopup(i: number) {
        markers[i].openPopup();
    }

    async function toggleMapFullScreen() {
        goto(`/map/trail/${trail.id!}`);
    }
</script>

<div
    class="trail-info-panel max-w-5xl mx-auto border border-input-border rounded-3xl overflow-x-hidden h-full"
>
    <div class="trail-info-panel-header sticky -top-44 z-10">
        <section class="relative h-80">
            <img class="w-full h-80" src={thumbnail} alt="" />
            <div
                class="absolute bottom-0 w-full h-1/2 bg-gradient-to-b from-transparent to-black opacity-50"
            ></div>
            <div
                class="flex absolute justify-between items-end w-full bottom-8 left-0 px-8 gap-y-4"
            >
                <div class="text-white">
                    <h4 class="text-4xl font-bold">
                        {trail.name}
                    </h4>
                    <div class="flex flex-wrap gap-x-8 gap-y-2 mt-4 mr-8">
                        {#if trail.location}
                            <h3 class="text-lg">
                                <i class="fa fa-location-dot mr-2"></i>
                                {trail.location}
                            </h3>
                        {/if}
                        <h3 class="text-lg">
                            <i class="fa fa-person-hiking mr-2"></i>
                            {$_(trail.difficulty ?? "?")}
                        </h3>
                    </div>
                </div>
                {#if $currentUser && $currentUser.id == trail.author}
                    <TrailDropdown {trail} {mode}></TrailDropdown>
                {/if}
            </div>
        </section>
        {#if mode == "overview"}
            <section
                class="grid grid-cols-2 sm:grid-cols-4 gap-y-4 justify-around py-8 border-b border-input-border"
            >
                <div class="flex flex-col items-center">
                    <span>{$_("distance")}</span>
                    <span class="font-semibold text-lg"
                        >{formatDistance(trail.distance)}</span
                    >
                </div>
                <div class="flex flex-col items-center">
                    <span>{$_("elevation-gain")}</span>
                    <span class="font-semibold text-lg"
                        >{formatElevation(trail.elevation_gain)}</span
                    >
                </div>
                <div class="flex flex-col items-center">
                    <span>{$_("est-duration")}</span>
                    <span class="font-semibold text-lg"
                        >{formatTimeHHMM(trail.duration)}</span
                    >
                </div>
                {#if trail.expand.category}
                    <div class="flex flex-col items-center">
                        <span>{$_("category")}</span>
                        <span class="font-semibold text-lg"
                            >{trail.expand.category.name}</span
                        >
                    </div>
                {/if}
            </section>
        {/if}
        {#if trail.tags && trail.tags.length > 0}
            <hr class="border-separator" />
            <section class="flex p-8 gap-4 text-gray-600">
                {#each trail.tags as tag}
                    <span class="py-2 px-4 border rounded-full">{tag}</span>
                {/each}
            </section>
            <hr class="border-separator" />
        {/if}
        <section class="trail-info-panel-tabs px-4 py-2 bg-background">
            <Tabs {tabs} bind:activeTab></Tabs>
        </section>
    </div>
    <section class="trail-info-panel-content px-8">
        <div
            class="grid grid-cols-1 mt-4 gap-8"
            class:md:grid-cols-[1fr_18rem]={mode == "overview"}
        >
            {#if activeTab == 0}
                <article class="text-justify whitespace-pre-line text-sm">
                    {trail.description}
                </article>
            {/if}
            {#if activeTab == 1}
                <ul>
                    {#each trail.expand.waypoints ?? [] as waypoint, i}
                        <li on:mouseenter={() => openMarkerPopup(i)}>
                            <WaypointCard {waypoint}></WaypointCard>
                        </li>
                    {/each}
                </ul>
            {/if}
            {#if activeTab == 2}
                <div
                    id="photo-gallery"
                    class="grid grid-cols-1 {mode == 'overview'
                        ? 'sm:grid-cols-2 md:grid-cols-3'
                        : ''} gap-4"
                >
                    <PhotoGallery
                        photos={trail.photos.map((p) => getFileURL(trail, p))}
                        bind:open={openGallery}
                    ></PhotoGallery>
                    {#each trail.photos ?? [] as photo, i}
                        <!-- svelte-ignore a11y-click-events-have-key-events -->
                        <!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
                        <img
                            class="rounded-xl cursor-pointer hover:scale-105 transition-transform"
                            on:click={() => openGallery(i)}
                            src={getFileURL(trail, photo)}
                            alt=""
                        />
                    {/each}
                </div>
            {/if}
            {#if activeTab == 3}
                <ul>
                    {#each trail.expand.summit_logs ?? [] as log}
                        <li><SummitLogCard {log}></SummitLogCard></li>
                    {/each}
                </ul>
            {/if}
            {#if mode == "overview"}
                <div class="relative">
                    <div class="rounded-xl h-72" id="map">
                        <div class="leaflet-top leaflet-right">
                            <button
                                class="leaflet-control fa fa-maximize rounded-full text-lg bg-white text-black px-[14px] py-2 hover:bg-gray-100"
                                style="cursor: pointer !important"
                                on:click={() => toggleMapFullScreen()}
                            ></button>
                        </div>
                    </div>
                </div>
            {/if}
        </div>
    </section>
</div>

<style>
    .trail-info-panel img {
        object-fit: cover;
    }
</style>
