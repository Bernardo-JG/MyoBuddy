# MyoBuddy

An EMG-based rehabilitation-assistance prototype that connects muscle-signal classification to a guided desktop exercise workflow. The application presents exercise steps, checks recognised movements, gives visual/audio feedback and records session reports. An Arduino sketch adds a serial-controlled four-servo demonstration arm.

Developed as a university group project in the context of stroke rehabilitation. Bernardo's main contribution was the machine-learning component for recognising movement from EMG and making those predictions usable by the desktop application.

## What is included

- A **PySide6 desktop interface** with English and Portuguese text, exercise videos, an EMG plot, patient/session selection and manual progression controls.
- Exercise sequences based on tasks such as drinking from a cup, using a spoon, reaching for a book and turning a door knob.
- **Threaded acquisition and inference**, with movement-completion events used to advance the expected exercise step.
- An offline training/evaluation script with signal features, temporal context, grouped cross-validation and movement-level metrics.
- A saved classifier artifact, sample EMG recordings and assembled exercise sequences for replay.
- Per-session Markdown reports and aggregate summaries/plots; optional PDF conversion code.
- Arduino commands for moving the demonstration arm through predefined exercise positions.

**The checked-in desktop acquisition path imports `mock_nidaqmx_module`, not the real NI-DAQmx driver.** It runs simulated or prerecorded signals. The NI-style channel configuration shows the intended acquisition interface, but changing a flag alone does not provide live hardware acquisition.

## From EMG to exercise feedback

```text
Simulated / recorded EMG → acquisition worker → processing worker
   → DC correction + filtering → window features + temporal context
   → saved model + smoothing → movement-completion event
   → expected exercise-step check → feedback / report / optional Arduino
```

### Processing and models

The implementation is configured around **four channels at 2,000 Hz**. The training entry point retains all four channels and includes Rest, Flexion, Extension, Pronation, Supination and Grasp_Power labels.

- DC-offset correction, a fourth-order **20–450 Hz Butterworth band-pass** and a **50 Hz notch**.
- **500-sample windows** (250 ms at 2 kHz), with **75% overlap** in offline training.
- Time-domain features, spectral features, frequency-band RMS and cross-channel relationships.
- Temporal context based on three previous windows, plus prediction smoothing with a default five-prediction history.
- Comparison of random forest, RBF SVM, logistic regression, gradient boosting and an MLP, each in a scaling/classification pipeline.
- `GroupKFold` validation using **trial-number groups**, not leave-one-subject-out validation. The number of folds equals the number of distinct trial groups.

The code also evaluates movement events using detection rate, false activations and latency, rather than relying only on window accuracy. The real-time worker maintains an adaptive DC estimate and uses predicted movement transitions to identify completed actions.

The intensity returned by the worker is based on signal RMS. Some application paths pass classifier confidence into the variable used for robot speed, so that control value should not be interpreted as calibrated muscle force.

## Run the desktop prototype

A Python environment compatible with the pinned packages is required; Python 3.11 or 3.12 is a practical starting point, but no complete environment validation is recorded here. The dependency snapshot is oriented toward Windows and includes both Qt bindings; **the main application uses PySide6**.

```bash
git clone https://github.com/Bernardo-JG/MyoBuddy.git
cd MyoBuddy
python -m venv .venv
```

Activate the virtual environment, then:

```bash
python -m pip install -r requirements.txt
python -m pip install seaborn antropy
python main_v1.py
```

`seaborn` and `antropy` are used by the source but missing from the checked-in requirements. Run from the repository root so the saved model and media resolve. Keep `best_temporal_emg_model.pkl` in that directory; it is loaded with joblib and requires compatible scikit-learn dependencies. Only load trusted model artifacts.

Start with a disposable patient name. Use the application's test-file controls to select a recording from `test_data/` or `exercise_sequences/`; `.npy` files are expected to represent channel-by-sample data. For example, the supplied cup sequence with rests contains four channels over **18.40 seconds**. Replay is an integration aid, not an independent patient-performance evaluation.

The default mock filename `emg_sequence_test.npy` is not included. When no recording is available, the mock module can generate synthetic data; explicitly select the desired file to know what the demonstration is processing.

### Optional Arduino demonstration

`arduino_commands.ino` uses the Arduino Servo library and attaches base, shoulder, elbow and gripper servos to pins **4, 5, 6 and 7**. Serial communication is **9,600 baud**, matching `BAUD_RATE` in `main_v1.py`.

The desktop defaults to `COM3`; adjust `SERIAL_PORT` for the intended device. Commands are newline-terminated frames of the form `<step_id:velocity>`, with a 0–255 velocity parameter controlling movement timing. The sketch uses predefined servo positions for each step. Test the sketch and mechanical travel separately before connecting it to the feedback workflow; this is demonstration hardware, not a validated patient-assistance controller.

### Reports

Reports are stored below the user-data location returned by `platformdirs.user_data_dir("StrokeRehabApp", "MioBuddies")`, in an `output` subdirectory. They include step timings, incorrect/weak/no-movement attempts and movement summaries. Aggregate reports and plots are generated by `report_generator.py`.

Optional PDF conversion uses Pandoc and XeLaTeX. The current prerequisite-check function references `pypandoc` without a module-level import, so that path needs a code correction before it can be relied on. Markdown reporting is the primary inspectable output.

## Offline training

`emg_classifier.py` expects a `user_defined_emg_data_4ch/` directory with EMG `.npy` files, matching label CSVs and optional `.npy.npz` metadata. Its loader derives labels and groups from recording filenames and associated files.

The full training directory is **not included** in this snapshot; the sample replay files are not a complete replacement. With a compatible labelled dataset configured:

```bash
python emg_classifier.py
```

The script writes results, plots and model artifacts, including `Models/best_temporal_emg_model.pkl`. The desktop reads the root-level `best_temporal_emg_model.pkl`, so replacing the runtime model is a separate, deliberate step.

## Repository map

| File / directory | Purpose |
| --- | --- |
| `main_v1.py` | Desktop UI, exercise progression, serial integration and session reporting |
| `emg_worker_threaded.py` | Acquisition/inference workers, adaptive offsets and movement tracking |
| `emg_classifier.py` | Offline preprocessing, features, training and evaluation |
| `mock_nidaqmx_module.py` | Simulated/prerecorded NI-style acquisition interface |
| `report_generator.py` | Markdown summaries, plots and optional PDF conversion |
| `arduino_commands.ino` | Four-servo demonstration-arm controller |
| `best_temporal_emg_model.pkl` | Saved runtime model and metadata |
| `test_data/`, `exercise_sequences/` | Recordings and constructed sequences for replay |
| `images/`, `sounds/`, `videos/` | UI and exercise media; some referenced videos are missing |
| `StrokeRehabApp.spec` | PyInstaller packaging configuration |

## Evaluation and implementation boundaries

The training source supports trial-grouped evaluation, per-class metrics, temporal smoothing and movement-event analysis. The repository does not include a complete, reproducible final metric table with its training dataset, so this README does not assign a headline accuracy to the shipped application.

Several details need alignment before treating a rerun as a validated end-to-end system:

- Offline spectral features use Welch estimation, while the runtime feature extractor uses FFT-based calculations. Feature semantics must match, not just feature count; the runtime currently pads/truncates mismatched vectors.
- Trial grouping is not participant-independent validation. Group identifiers and temporal smoothing across concatenated recordings need care when interpreting results.
- The saved pipeline handling in the training loop should be reviewed to ensure the final exported model has been refitted consistently on the intended training data.
- The classifier uses `Grasp_Power`, while the cup exercise expects `Grasp`, which can prevent that step from advancing automatically.
- Several configured exercise videos are absent from the repository snapshot.

The project demonstrates EMG processing and application integration. It does not establish clinical rehabilitation benefit, generalisation to stroke patients or suitability for unsupervised physical assistance.
