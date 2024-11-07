![routes](https://github.com/user-attachments/assets/dc5fb960-5015-4c92-9564-57dc61f11ce2)

import pandas as pd
import geopandas as gpd
import osmnx as ox
import matplotlib.pyplot as plt
import gpxpy
import os
from geopy.distance import geodesic
from shapely.geometry import LineString
from matplotlib.patches import Rectangle
from matplotlib.lines import Line2D


# Folder containing GPX files
gps_file_path =   # Path to the GPX files

# Load the GPS data for restaurants
gps_data_Pre_restaurants = pd.read_csv() # Path to the GPS locations Restaurants
gps_gdf_Pre_restaurants = gpd.GeoDataFrame(gps_data_Pre_restaurants,
                                           geometry=gpd.points_from_xy(gps_data_Pre_restaurants.longitude,
                                                                       gps_data_Pre_restaurants.latitude))
# Load the GPS data for farms
gps_data_farms = pd.read_csv() # Path to the GPS locations Farms
gps_gdf_farms = gpd.GeoDataFrame(gps_data_farms,
                                 geometry=gpd.points_from_xy(gps_data_farms.longitude, gps_data_farms.latitude))
gps_gdf_farms.set_crs(epsg=4326, inplace=True)

# Define folder paths for each mode of transport
folder_paths = {
    'bike': '', # Path to the GPX files Bike
    'foot': '', # Path to the GPX files foot
    'boda': '', # Path to the GPX files boda
    'car': '' # Path to the GPX files car
}

# Color mapping for each mode of transport
colors = {
    'foot': '#add8e6',
    'bike': 'darkturquoise',
    'boda': '#1e90ff',
    'car': '#00008b'
}



# Initialize lists to hold all GeoDataFrames
gdfs = []
route_distances = []  # List to hold route and distance information

# Read the GPS locations CSV and correct latitude and longitude
gps_data = pd.read_csv(gps_file_path)

# Check for the presence of columns and rename if necessary
gps_data = gps_data.rename(columns={"long": "longitude", "lat": "latitude"})  # Correct column names

# Loop through all GPX files in the folder
# Loop through each mode of transport folder
for mode, folder_path in folder_paths.items():
    for filename in os.listdir(folder_path):
        if filename.endswith(".gpx"):
            gpx_file_path = os.path.join(folder_path, filename)
            with open(gpx_file_path, 'r') as gpx_file:
                gpx = gpxpy.parse(gpx_file)

                # Check if GPX data is properly parsed
                if not gpx.tracks:
                    print(f"No tracks found in {filename}")
                    continue

                track_points = []
                total_distance = 0

                # Extract track points from the current GPX file
                for track in gpx.tracks:
                    for segment in track.segments:
                        segment_points = segment.points
                        if not segment_points:
                            print(f"No points found in track {track.name} of file {filename}")
                            continue

                        previous_point = None
                        for point in segment_points:
                            track_points.append((point.longitude, point.latitude))

                            # Distance calculation
                            current_point = (point.latitude, point.longitude)
                            if previous_point:
                                distance = geodesic(previous_point, current_point).meters
                                total_distance += distance
                            previous_point = current_point

                # Store the route and distance information
                rounded_distance = round(total_distance)
                print(f"{filename}; Distance: {rounded_distance} m; Mode: {mode}")

                # Convert track points to a LineString and store with color
                if track_points:
                    track_line = LineString(track_points)
                    track_gdf = gpd.GeoDataFrame({'geometry': [track_line], 'color': [colors[mode]]}, crs="EPSG:4326")
                    gdfs.append(track_gdf)


# Get the road network and city boundaries for Kampala
place_name = "Kampala, Uganda"
graph = ox.graph_from_place(place_name, network_type='all')
gdf_edges = ox.graph_to_gdfs(graph, nodes=False)
city_boundary = ox.geocode_to_gdf(place_name)

# Function to add a scale bar that scales with the plot
def add_scale_bar(ax, location=(32.5, 0.2), length_in_km=5, linewidth=3, color="black"):
    bar_x, bar_y = location  # Longitude, Latitude

    # Calculate the longitude difference corresponding to the desired length in kilometers
    start_point = (bar_y, bar_x)  # (latitude, longitude)
    end_point = geodesic(kilometers=length_in_km).destination(start_point, 90)  # 90 degrees (East)
    end_longitude = end_point.longitude

    # Calculate the width of the scale bar in longitude terms
    length = end_longitude - bar_x

    # Add the black part of the scale bar (5 km)
    ax.add_patch(Rectangle((bar_x, bar_y), length, 0.002, color='black', lw=0))

    # Add labels for 0 km and 5 km
    ax.text(bar_x, bar_y - 0.005, '0 km', color=color, fontsize=12, ha='center')
    ax.text(bar_x + length, bar_y - 0.005, '5 km', color=color, fontsize=12, ha='center')


# Create a figure
fig, ax1 = plt.subplots(figsize=(9, 9))  # One panel (ax1)

# Panel 1: Map Plot
add_scale_bar(ax1)

# Plot roads with transparency so that they appear in the background
gdf_edges.plot(ax=ax1, color='#A9A9A9', linewidth=0.5, alpha=0.3)

# Plot city boundaries
city_boundary.plot(ax=ax1, edgecolor='black', facecolor='none', linewidth=1)

# Plot the GPX data as lines in red
for gdf in gdfs:
    gdf.plot(ax=ax1, color=gdf['color'].iloc[0], linewidth=2)

# # Create custom legend handles
legend_handles = [
     Line2D([0], [0], color= 'limegreen', marker='o', markersize=8, label='Restaurant Locations', markeredgecolor='black', markeredgewidth=1, linestyle="none"),
     Line2D([0], [0], color='violet', marker='o', markersize=8, label='Farm Locations', markeredgecolor='black', markeredgewidth=1, linestyle="none"),
     Line2D([0], [0], color=colors['foot'], linewidth=2, label='Foot'),
     Line2D([0], [0], color=colors['bike'], linewidth=2, label='Bike'),
     Line2D([0], [0], color=colors['boda'], linewidth=2, label='Boda'),
     Line2D([0], [0], color=colors['car'], linewidth=2, label='Car'),
]

# Add legend to the plot
ax1.legend(handles=legend_handles,loc='upper left', frameon=False)

# Plot the farm and restaurant locations with different markers
ax1.scatter(gps_gdf_Pre_restaurants.geometry.x, gps_gdf_Pre_restaurants.geometry.y, color= 'limegreen', s=20, marker='o',
            zorder=5, edgecolor='black', linewidth=1)
ax1.scatter(gps_gdf_farms.geometry.x, gps_gdf_farms.geometry.y, color= 'violet', s=20, marker='o', zorder=6,
            edgecolor='black', linewidth=1)

# Remove the axis borders
ax1.spines['top'].set_visible(False)
ax1.spines['right'].set_visible(False)
ax1.spines['bottom'].set_visible(False)
ax1.spines['left'].set_visible(False)

# Remove x and y ticks (axis numbers)
ax1.set_xticks([])
ax1.set_yticks([])

# Remove latitude and longitude labels
ax1.set_xlabel('')
ax1.set_ylabel('')

#ax1.legend()
plt.savefig('') # path to where you want the file to be saved
plt.show()
