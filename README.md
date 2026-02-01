import React, { useEffect, useState } from 'react';
import { View, Text, Button, TouchableOpacity, Linking, ScrollView } from 'react-native';
import MapView, { Marker } from 'react-native-maps';
import * as Location from 'expo-location';

import { initializeApp } from 'firebase/app';
import { getDatabase, ref, set, onValue, update } from 'firebase/database';

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_DOMAIN",
  databaseURL: "YOUR_DB_URL",
  projectId: "YOUR_PROJECT_ID",
};

const app = initializeApp(firebaseConfig);
const db = getDatabase(app);

const BASE_FARE = 3;
const PER_MILE = 2.25;

const distanceMiles = (a, b) => {
  const R = 3958.8;
  const dLat = (b.latitude - a.latitude) * Math.PI / 180;
  const dLon = (b.longitude - a.longitude) * Math.PI / 180;
  const lat1 = a.latitude * Math.PI / 180;
  const lat2 = b.latitude * Math.PI / 180;

  const h =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) ** 2;

  return R * (2 * Math.atan2(Math.sqrt(h), Math.sqrt(1 - h)));
};

export default function App() {
  const [mode, setMode] = useState(null);
  const [location, setLocation] = useState(null);
  const [rides, setRides] = useState({});
  const driverId = "driver_001";

  useEffect(() => {
    Location.requestForegroundPermissionsAsync().then(() =>
      Location.watchPositionAsync(
        { accuracy: Location.Accuracy.High, distanceInterval: 5 },
        loc => setLocation(loc.coords)
      )
    );

    onValue(ref(db, 'rides'), snap => setRides(snap.val() || {}));
  }, []);

  const requestRide = () => {
    const id = `ride_${Date.now()}`;
    const drop = {
      latitude: location.latitude + 0.01,
      longitude: location.longitude + 0.01
    };

    const miles = distanceMiles(location, drop);
    const fare = (BASE_FARE + miles * PER_MILE).toFixed(2);

    set(ref(db, `rides/${id}`), {
      pickup: location,
      dropoff: drop,
      status: 'requested',
      fare
    });
  };

  const navigate = (coords) => {
    Linking.openURL(
      `https://www.google.com/maps/dir/?api=1&destination=${coords.latitude},${coords.longitude}`
    );
  };

  if (!mode) {
    return (
      <View style={{ flex: 1, justifyContent: 'center' }}>
        <Button title="Driver" onPress={() => setMode('driver')} />
        <Button title="Rider" onPress={() => setMode('rider')} />
      </View>
    );
  }

  return (
    <View style={{ flex: 1 }}>
      {location && (
        <MapView
          style={{ flex: 1 }}
          initialRegion={{
            latitude: location.latitude,
            longitude: location.longitude,
            latitudeDelta: 0.05,
            longitudeDelta: 0.05
          }}
        >
          {Object.entries(rides).map(([id, r]) => (
            <Marker key={id} coordinate={r.pickup} title={`$${r.fare}`} />
          ))}
        </MapView>
      )}

      {mode === 'rider' && (
        <Button title="Request Ride" onPress={requestRide} />
      )}

      {mode === 'driver' && (
        <ScrollView style={{ maxHeight: 250 }}>
          {Object.entries(rides).map(([id, r]) => (
            <View key={id} style={{ padding: 8 }}>
              <Text>{id} — ${r.fare} — {r.status}</Text>

              {r.status === 'requested' && (
                <Button title="Accept"
                  onPress={() =>
                    update(ref(db, `rides/${id}`), {
                      status: 'inProgress',
                      driver: driverId
                    })
                  }
                />
              )}

              {r.status === 'inProgress' && (
                <>
                  <Button title="Navigate to Pickup" onPress={() => navigate(r.pickup)} />
                  <Button title="Complete Ride"
                    onPress={() =>
                      update(ref(db, `rides/${id}`), { status: 'completed' })
                    }
                  />
                </>
              )}
            </View>
          ))}
        </ScrollView>
      )}
    </View>
  );
}
