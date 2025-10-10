# 🌾 AgroVision: Real-Time Crop Health Monitoring System

**AgroVision** is an AI-driven system designed to monitor crop health in real time using aerial imagery and deep learning.
It simulates UAV (Unmanned Aerial Vehicle) operations that capture agricultural field images, perform onboard inference, and stream results to a central FastAPI backend and React dashboard for visualization.

The goal: enable farmers and researchers to detect early signs of crop stress, weeds, and anomalies, ensuring timely interventions and better yield outcomes.

![System Dashboard](/Assets/AgroVision_Interface.png)

## System Architecture

The system processes high-resolution imagery from unmanned aerial vehicles (UAVs) equipped with multispectral cameras capturing both RGB and near-infrared (NIR) data. Apache Kafka streams this raw data in real time, simulating multiple UAVs as producers sending images to a single consumer. Raw images, which can reach 200 MB, are downsized to approximately 10 MB for efficiency. Historical data is stored in MongoDB, while processed images are saved in a MinIO bucket-style storage system.

> The system outputs three types of prepared images:

- RGB images for visual inspection
- NDVI (Normalized Difference Vegetation Index) images to assess crop health
- Prediction masks overlaid on RGB images, highlighting stress types (e.g., nutrient deficiency, drydown, or water stress)

![System Architecture](https://myoctocat.com/assets/images/base-octocat.svg)

## Project Structure

```
.
├── backend/           # FastAPI backend for streaming, MinIO, Kafka integration
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── main.py
│   └── requirements.txt
├── frontend/          # React frontend for visualization
│   ├── Dockerfile
│   ├── package-lock.json
|   ├── package.json
│   ├── public/
│   └── src/
├── uav-producer/      # UAV producer service (model inference, MinIO/Kafka upload)
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── model.tflite
│   ├── producer.py
│   └── requirements.txt
├── data/
├── create_yml.py      # Script to generate docker-compose files with multiple UAVs
├── docker-compose.yml # Main Compose file (edit or generate as needed)
├── ...
```

## Folder Descriptions

- **backend/**: FastAPI app that consumes Kafka messages, serves WebSocket/API for frontend, and manages MinIO bucket.
- **frontend/**: React app for real-time visualization of UAV images, predictions, and logs.
- **uav-producer/**: Simulates UAVs. Reads images, runs ML inference, uploads to MinIO, and sends Kafka messages.
- **data/**: data from agriculture-vision-2021 (raw).
- **create_yml.py**: Python script to generate a `docker-compose` file with any number of UAV producers.

---

## Dataset & Preprocessingn
The system uses the Agriculture-Vision 2021 dataset:
🔗 [https://www.agriculture-vision.com/agriculture-vision-2021/dataset-2021](https://www.agriculture-vision.com/agriculture-vision-2021/dataset-2021)

The focus was on crop health, selecting five key stress classes: nutrient deficiency, drydown, water, planter skip, and weed cluster. Remaining classes (double plant, endrow, storm damage, waterway) were grouped into a sixth “none-selected” class for non-health-related conditions. All images and corresponding masks were resized to 256×256 pixels, with roughly 5,824 samples distributed evenly across the six classes to ensure balance.

The dataset was split into training (75%), validation (5%), and test (20%) subsets, maintaining class balance:

- **Training**: 4,369 samples

- **Validation**: 291 samples

- **Test**: 1,164 samples

This preparation ensures the model focuses on classes directly impacting crop health.

## ML Model Design

The model used is a dual-output U-Net architecture optimized for both efficiency and accuracy. The model takes 256×256 NDVI (Normalized Difference Vegetation Index) images as input, enhancing its ability to detect vegetation stress. It produces two simultaneous outputs:

- Segmentation masks for pixel-level identification of affected areas

- Multiclass predictions for categorizing stress types

![Model Design](https://myoctocat.com/assets/images/base-octocat.svg)

The model achieved 95% accuracy and a 70% Dice Coefficient, with performance evaluated using a confusion matrix to assess class-wise prediction reliability.

![Confusion Matrix](https://myoctocat.com/assets/images/base-octocat.svg)

## Quick Start Guide

### 1. Generate a Docker Compose File with Multiple UAV Producers

To create a Compose file with multiple UAV producers (e.g., 5 UAVs):

```sh
python create_yml.py uav-producer 5 docker-compose.yml docker-compose.multi.yml
```

- This will create `docker-compose.multi.yml` with services: `uav-producer-1`, `uav-producer-2`, ..., `uav-producer-5`.
- Each producer gets a unique `UAV_ID` (e.g., "01", "02", ...).

### 2. Prepare Data

- Place your UAV input data (GeoTIFF bands, etc.) in the `data/` directory.
- Each producer expects its files under `/data/<UAV_ID>` inside the container.

### 3. Build and Launch the Stack

To launch the full system (backend, frontend, MinIO, Kafka, and all producers):

```sh
docker compose -f docker-compose.multi.yml up --build
```

- The first launch may take a few minutes to build images and initialize services.
- The backend will be available at [http://localhost:8000](http://localhost:8000)
- The frontend will be available at [http://localhost:3000](http://localhost:3000)
- MinIO console: [http://localhost:9001](http://localhost:9001) (login: `minioadmin`/`minioadmin`)
- Kafka UI: [http://localhost:8080](http://localhost:8080)

### 4. Stopping the System

To stop and remove all containers:

```sh
docker compose -f docker-compose.multi.yml down
```

---

## Notes

- The MinIO bucket `uav-images` is created and made public automatically by the `minio-init` service.
- Each UAV producer uploads images and predictions to MinIO and sends metadata to Kafka.
- The backend consumes Kafka messages and serves them to the frontend via WebSocket.
- The frontend displays real-time UAV imagery, predictions, and logs.

---

## Customization

- To change the number of UAVs, rerun `create_yml.py` with a different count and restart Docker Compose.
- To use the default single-producer setup, you can use the provided [`docker-compose.yml`](docker-compose.yml) directly.

---

## Troubleshooting

- Ensure ports 8000, 3000, 9000, 9001, and 8080 are free.
- If you change the number of UAVs, always regenerate the compose file and restart the stack.
- Check logs with `docker compose logs -f`.
