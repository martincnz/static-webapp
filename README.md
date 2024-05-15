import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.Semaphore;
import java.util.concurrent.atomic.AtomicInteger;

class Horse implements Runnable {
    private final String name;
    private final int speed;
    private final int resistance;
    private final AtomicInteger position = new AtomicInteger(0);
    private final Random random = new Random();
    private final PowerUpArea powerUpArea;
    private final Race race;

    public Horse(String name, int speed, int resistance, PowerUpArea powerUpArea, Race race) {
        this.name = name;
        this.speed = speed;
        this.resistance = resistance;
        this.powerUpArea = powerUpArea;
        this.race = race;
    }

    public String getName() {
        return name;
    }

    public int getPosition() {
        return position.get();
    }

    @Override
    public void run() {
        while (position.get() < 1000) {
            // Advancement Stage
            int advancement = speed * (random.nextInt(10) + 1);
            position.addAndGet(advancement);
            System.out.println(name + " advances to " + position.get() + " meters.");

            // Check for power-up
            if (powerUpArea.checkAndApplyPowerUp(position.get())) {
                System.out.println(name + " activated the power-up!");
                position.addAndGet(100);
            }

            // Check if race is over
            if (race.isRaceOver()) break;

            // Waiting Stage
            int waitTime = random.nextInt(5) + 1 - resistance;
            if (waitTime > 0) {
                try {
                    Thread.sleep(waitTime * 1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
        race.finish(this);
    }
}

class PowerUpArea implements Runnable {
    private volatile int position = new Random().nextInt(950) + 50;
    private final Object lock = new Object();

    public int getPosition() {
        return position;
    }

    public boolean checkAndApplyPowerUp(int horsePosition) {
        synchronized (lock) {
            if (horsePosition >= position && horsePosition < position + 50) {
                position = new Random().nextInt(950) + 50;
                try {
                    Thread.sleep(7000); // Simulate power-up activation
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return true;
            }
        }
        return false;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (lock) {
                position = new Random().nextInt(950) + 50;
            }
            try {
                Thread.sleep(15000); // Refresh power-up position every 15 seconds
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}

class Race {
    private final List<Horse> horses = new ArrayList<>();
    private final Semaphore raceSemaphore = new Semaphore(3);
    private final AtomicInteger finishedCount = new AtomicInteger(0);

    public Race(int numberOfHorses) {
        PowerUpArea powerUpArea = new PowerUpArea();
        new Thread(powerUpArea, "PowerUpThread").start();

        Random random = new Random();
        for (int i = 0; i < numberOfHorses; i++) {
            String name = "Horse" + (i + 1);
            int speed = random.nextInt(3) + 1;
            int resistance = random.nextInt(3) + 1;
            horses.add(new Horse(name, speed, resistance, powerUpArea, this));
        }

        System.out.println("Horses in the race:");
        for (Horse horse : horses) {
            System.out.println(horse.getName() + " - Speed: " + speed + ", Resistance: " + resistance);
        }
    }

    public void startRace() {
        int processors = Runtime.getRuntime().availableProcessors();

        if (processors > 4) {
            horses.forEach(horse -> Thread.ofVirtual().start(horse));
        } else {
            horses.forEach(horse -> new Thread(horse).start());
        }
    }

    public void finish(Horse horse) {
        int finished = finishedCount.incrementAndGet();
        System.out.println(horse.getName() + " has finished the race!");

        if (finished >= 3) {
            System.out.println("Race is over. Top 3 horses have finished.");
            System.exit(0);
        }
    }

    public boolean isRaceOver() {
        return finishedCount.get() >= 3;
    }

    public static void main(String[] args) {
        int numberOfHorses = 5; // Example input
        Race race = new Race(numberOfHorses);
        race.startRace();
    }
}