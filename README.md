# Health
### This package is a bug fixing of ticket https://github.com/cph-cachet/flutter-plugins/issues/1072

My native iOS code not good but at least solved the problem.
Please contribute to this plugin to make it can reuse in the future.

# USING
Download and put it in project same level as lib:
<img width="261" alt="image" src="https://github.com/user-attachments/assets/73b16cba-f473-4630-a87a-4bc9902beb94">


Create health_service.dart and copy-paste this code:
```
import 'dart:io';

import 'package:flutter_bugfender/flutter_bugfender.dart';
import 'package:health/health.dart';
import 'package:walk2bfit/core/extension/date_time_extension.dart';
import 'package:walk2bfit/core/extension/num_extension.dart';
import 'package:walk2bfit/core/utils/logger.dart';
import 'package:walk2bfit/feature/activity/data/model/activity_model.dart';


class HealthServices {
  HealthServices._();

  static HealthServices get instance => HealthServices._();

  // create a HealthFactory for use in the app
  HealthFactory health = HealthFactory(useHealthConnectIfAvailable: true);

  List<HealthDataType> get types =>
      Platform.isIOS ? [
        HealthDataType.STEPS,
        HealthDataType.ACTIVE_ENERGY_BURNED,
        HealthDataType.EXERCISE_TIME,
        HealthDataType.DISTANCE_WALKING_RUNNING,
      ] : [
        HealthDataType.STEPS,
        HealthDataType.ACTIVE_ENERGY_BURNED,
        HealthDataType.HEART_RATE,
        // HealthDataType.MOVE_MINUTES,
        HealthDataType.DISTANCE_DELTA,
        HealthDataType.WORKOUT,
      ];

  Future<void> requestAuthorization() async {
    FlutterBugfender.log("HealthService: requestAuthorization");
    await health.requestAuthorization(types);
  }

  void logStepsRecordIf30Days() async {
    for (int i = 0; i <= 10; i++) {
      DateTime startDatePoint = DateTime.now()
          .subtract(Duration(days: i))
          .startOfTheDay();
      DateTime endDatePoint = DateTime.now()
          .subtract(Duration(days: i))
          .endOfTheDay();
      int? totalStepCount = await health.getTotalStepsInInterval(
          startDatePoint, endDatePoint);
      FlutterBugfender.log(
          'Fetched STEP: $totalStepCount | from: $startDatePoint  - to: $endDatePoint');

      Logger.i(
          'Fetched STEP:  $totalStepCount | from: $startDatePoint  - to: $endDatePoint');
    }
  }

  void fetch7DaysData({
    required DateTime start,
    required DateTime end,
  }) async {
    try {

      int totalSteps = 0;
      double totalDistance = 0;
      int totalCalories = 0;
      double totalMinutes = 0;

      List<HealthDataPoint> healthData = await health.getHealthDataFromTypes(
        start,
        end,
        types,
      );

      for (var health in healthData) {
        switch (health.type) {
          case HealthDataType.STEPS:
            totalSteps = (health.value as NumericHealthValue).numericValue.toInt();
            print("totalSteps count: $totalSteps");
            break;
          case HealthDataType.ACTIVE_ENERGY_BURNED:
            totalCalories = (health.value as NumericHealthValue).numericValue.toInt();
            print("totalCalories count: $totalCalories");
            break;
          case HealthDataType.DISTANCE_WALKING_RUNNING:
            totalDistance = (health.value as NumericHealthValue).numericValue.toDouble().meterToMiles;
            print("totalDistance count: $totalDistance");
            break;
          case HealthDataType.EXERCISE_TIME:
            totalMinutes = (health.value as NumericHealthValue).numericValue.toDouble();
            print("totalMinutes count: $totalMinutes");
            break;
          default:
            break;
        }
      }

      ActivityModel resultActivity = ActivityModel(
          steps: totalSteps,
          distance: totalDistance.meterToMiles,
          calorie: totalCalories.toInt(),
          time: totalMinutes.toInt(),
          activityFromDate: start.yMdHmS,
          activityToDate: end.yMdHmS
      );

      Logger.i('Fetched Merged Activity: $resultActivity}');
    } catch (error, stackTrace) {
      Logger.e(
        'Failed to fetch health data: $error',
        stackTrace: stackTrace,
      );
      FlutterBugfender.log('Fetched Health Exception: $error');
    }
  }

  Future<ActivityModel> fetchHealthData({
    required DateTime start,
    required DateTime end,
  }) async {
    try {
      // requesting access to the data types before reading them
      bool requested = await health.requestAuthorization(types);

      if (!requested) {
        Logger.e('User deny health permission');
        return ActivityModel();
      }


      // int totalSteps = await health.getTotalStepsInInterval(start, end, dayInterval) ?? 0;


      int totalSteps = 0;
      double totalDistance = 0;
      int totalCalories = 0;
      // double totalMinutes = 0;

      List<HealthDataPoint> stepsData = await health.getHealthDataFromTypes(
        start,
        end,
        [HealthDataType.STEPS],
      );
      List<HealthDataPoint> caloriesData = await health.getHealthDataFromTypesCalories(
        start,
        end,
        [HealthDataType.ACTIVE_ENERGY_BURNED],
      );
      List<HealthDataPoint> distanceData = await health.getHealthDataFromTypesDistance(
        start,
        end,
        [HealthDataType.DISTANCE_WALKING_RUNNING],
      );
      // List<HealthDataPoint> exerciseData = await health.getHealthDataFromTypesExercise(
      //   start,
      //   end,
      //   [HealthDataType.EXERCISE_TIME],
      // );

      List<HealthDataPoint> allData = [...stepsData, ...caloriesData, ...distanceData];

      for (var health in allData) {
        switch (health.type) {
          case HealthDataType.STEPS:
            totalSteps = (health.value as NumericHealthValue).numericValue.toInt();
            break;
          case HealthDataType.ACTIVE_ENERGY_BURNED:
            totalCalories = (health.value as NumericHealthValue).numericValue.toInt();
            break;
          case HealthDataType.DISTANCE_WALKING_RUNNING:
            totalDistance = (health.value as NumericHealthValue).numericValue.toDouble().meterToMiles;
            break;
          // case HealthDataType.EXERCISE_TIME:
          //   totalMinutes = (health.value as NumericHealthValue).numericValue.toDouble();
          //   print("totalMinutes count: $totalMinutes");
          //   break;
          default:
            break;
        }
      }


      // double totalDistance = await health.getTotalDistanceInInterval(
      //     start, end) ?? 0;
      // double totalCalories = await health.getTotalCaloriesInInterval(
      //     start, end) ?? 0;
      double totalMinutes = await health.getTotalMinutesInInterval(
          start, end) ?? 0;

      ActivityModel resultActivity = ActivityModel(
          steps: totalSteps,
          distance: totalDistance,
          calorie: totalCalories.toInt(),
          time: totalMinutes.toInt(),
          activityFromDate: start.yMdHmS,
          activityToDate: end.yMdHmS
      );


      Logger.i('Fetched Merged Activity: $resultActivity}');
      FlutterBugfender.log('Fetched Merged Activity: $resultActivity}');

      return resultActivity;
    } catch (error, stackTrace) {
      Logger.e(
        'Failed to fetch health data: $error',
        stackTrace: stackTrace,
      );
      FlutterBugfender.log('Fetched Health Exception: $error');
      return ActivityModel();
    }
  }


}

```
dayInterval = 1, if you want the interval by hour or second please adjust in native code.



