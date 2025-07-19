package my;


import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import tpoints.me.service.communication.CommunicationInterface;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Slf4j
@Component
public class UZTicketMonitor {

    @Autowired
    CommunicationInterface communicationInterface;

    private final WebClient webClient;

    public UZTicketMonitor() {
        this.webClient = WebClient.builder()
                .baseUrl("https://app.uz.gov.ua/api/v3/trips")
                .defaultHeader("Authorization", "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
                        + "wI4jtNchCczlcrnoKV5OaxfIVQu5siVQH2xt4LJpEBo")
                .defaultHeader("x-client-locale", "uk")
                .defaultHeader("x-session-id", "17396ad7-8115-4f7f-82c0-73ce9b1a86fd")
                .defaultHeader("x-user-agent", "UZ/2 Web/1 User/78256")
                .defaultHeader("Accept", "application/json")
                .defaultHeader("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)")
                .build();
    }

    @PostConstruct
    @Scheduled(fixedRate = 60000)
    public void fetchTrainsOnce() {
        String stationFromId = "2218000";
        String stationToId = "2210700";
        String date = "2025-07-06";

        webClient.get()
                .uri(uriBuilder -> uriBuilder
                        .queryParam("station_from_id", stationFromId)
                        .queryParam("station_to_id", stationToId)
                        .queryParam("with_transfers", "0")
                        .queryParam("date", date)
                        .build())
                .retrieve()
                .bodyToMono(String.class)
                .map(this::extractTrainsWithSeats)
                .doOnError(error -> log.error("❌ Error data requesting: {}", error.getMessage()))
                .subscribe(result -> {
                    if (result.isEmpty()) {
                        log.info("😴 No available trains found.");
                    } else {
                        log.info("🎉 Found trains with the seats available:\n{}", result);
                        communicationInterface.sendText("🎉 Found trains with the seats available:\n" + "https://booking.uz.gov.ua/search-trips/2218000/2210700/list?startDate=2025-07-06");
                    }
                });
    }

    private String extractTrainsWithSeats(String response) {
        StringBuilder result = new StringBuilder();
        Pattern pattern = Pattern.compile("\"free_seats\"\\s*?:\\s*?([1-9][0-9]*)", Pattern.DOTALL);
        Matcher matcher = pattern.matcher(response);

        while (matcher.find()) {
            int seats = Integer.parseInt(matcher.group(1));
            if (seats > 0) {
                result.append(matcher.group()).append("\n---\n");
            }
        }

        return result.toString();
    }
}
