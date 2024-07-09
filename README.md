package com.aa.hmtautomation.eligibility.service.impl;

import com.aa.hmtautomation.DbCodesCachingUtil;
import com.aa.hmtautomation.db.entity.ClassOfServiceMappings;
import com.aa.hmtautomation.db.entity.CountryAirportCode;
import com.aa.hmtautomation.db.repository.ClassOfServiceMappingsRepository;
import com.aa.hmtautomation.ApplicationConstants;
import com.aa.hmtautomation.ApplicationUtils;
import com.aa.hmtautomation.db.entity.Eligibility;
import com.aa.hmtautomation.db.entity.EligibilityId;
import com.aa.hmtautomation.db.repository.EligibilityRepository;
import com.aa.hmtautomation.db.repository.OfferDetailsRepository;
import com.aa.hmtautomation.eligibility.model.Pax;
import com.aa.hmtautomation.eligibility.service.EligibilityService;
import com.aa.hmtautomation.eligibility.model.HMTAutomationPnr;
import com.aa.hmtautomation.eventprocessor.enums.TierStatusEnum;
import com.aa.hmtautomation.exception.HMTAutomationException;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.Arrays;
import java.sql.Timestamp;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.stream.Stream;

@Slf4j
@Service
public class MealEligibilityServiceImpl implements EligibilityService {

    private static final List<String> CLASS_OF_SERVICE = Arrays.asList("BUSINESS CLASS", "FIRST CLASS");

    public static final String AUTOMATED_DIGITAL_MEALS= "AUTOMATED_DIGITAL_MEALS";
    
    @Autowired
    private EligibilityRepository  eligibilityRepository;

    @Autowired
    private OfferDetailsRepository offerDetailsRepository;

    @Autowired
    private ClassOfServiceMappingsRepository classOfServiceMappingsRepository;

    @Autowired
    private DbCodesCachingUtil dbCodesCachingUtil;

    @Override
    public void checkEligibility(HMTAutomationPnr hmtAutomationPnrEvent) {
        log.info("TransactionId: {} ClassName: {} method: {} hmtAutomationPnrEvent: {}", hmtAutomationPnrEvent.getTransactionId(),
                this.getClass().getSimpleName(), "processHotelOrMealEvent", hmtAutomationPnrEvent);
       checkEligibilityCriteriaForMealsIssuance(hmtAutomationPnrEvent);
    }

    private void checkEligibilityCriteriaForMealsIssuance(HMTAutomationPnr hmtAutomationPnrEvent) {
        log.info("TransactionId: {} ClassName: {} method: {} hmtAutomationPnrEvent: {}", hmtAutomationPnrEvent.getTransactionId(),
                this.getClass().getSimpleName(), "checkEligibilityCriteriaForMealsIssuance", hmtAutomationPnrEvent);

        StringBuilder mealIneligibleReasons = new StringBuilder();
        boolean isPnrLiesInTopTierCategory = isPnrLiesInTopTierCategory(hmtAutomationPnrEvent.getPaxList(), hmtAutomationPnrEvent.getTransactionId());
        boolean isPnrLiesInPremiumCabinCategory = isPnrLiesInPremiumCabinCategory(hmtAutomationPnrEvent.getPaxList(),
                hmtAutomationPnrEvent.getTransactionId());
        boolean isPnrOfTypeNonAADV = isPnrOfTypeNonAADV(hmtAutomationPnrEvent.getPaxList(), hmtAutomationPnrEvent.getTransactionId());
        //mealIneligibleReasons.append(isPnrOfTypeNonAADV ? "Pnr is not eligible for meals as it is of type NonAADV.\n" : "");

        mealIneligibleReasons.append(!(isPnrLiesInTopTierCategory || isPnrLiesInPremiumCabinCategory || isPnrOfTypeNonAADV) ?
                "Pnr is not eligible for meals as it doesn't lie in top tier or premium cabin category or NonAADV customer.\n" : "");

        boolean isPnrOfTypeNonRev = isPnrOfTypeNonRev(hmtAutomationPnrEvent.getPaxList(), hmtAutomationPnrEvent.getTransactionId());
        mealIneligibleReasons.append(isPnrOfTypeNonRev ? "Pnr is not eligible for meals as it is of type non revenue.\n" : "");

        boolean isPnrOfTypeStandBy = isPnrOfTypeStandBy(hmtAutomationPnrEvent.getPaxList(), hmtAutomationPnrEvent.getTransactionId());
        mealIneligibleReasons.append(isPnrOfTypeStandBy ? "Pnr is not eligible for meals as it is of type standBy.\n" : "");

        mealIneligibleReasons.append(hmtAutomationPnrEvent.getIsAPass() ? "Pnr is not eligible for meals as it is of type apass.\n" : "");

        mealIneligibleReasons.append(hmtAutomationPnrEvent.getPortOfAccomodation().equalsIgnoreCase(hmtAutomationPnrEvent.
                getDestinationStation()) ? "Pnr is not eligible for meals as port of accommodation is same as destination station.\n" : "");

        boolean isMealIssuedOnPnr = offerDetailsRepository.isMealIssuedOnPnr(hmtAutomationPnrEvent.getFlightKey().getFltNum(),
                hmtAutomationPnrEvent.getFlightKey().getDepSta(), StringUtils.replace(hmtAutomationPnrEvent.getFlightKey().
                        getFltOrgDate(), "-", ""), hmtAutomationPnrEvent.getRecLoc());

        mealIneligibleReasons.append(isMealIssuedOnPnr ? "Meal is already issued on pnr.\n" : "");

        boolean isAutomatedDigitalMealsEnabled = isAutomatedDigitalMealsEnabled(hmtAutomationPnrEvent.getPortOfAccomodation(),
                hmtAutomationPnrEvent.getTransactionId());

        mealIneligibleReasons.append(!isAutomatedDigitalMealsEnabled ? "Port of Accommodation " + hmtAutomationPnrEvent.getPortOfAccomodation() + " is not eligible for Automated Digital Meal.\n" : "");

        Eligibility eligibility = eligibilityRepository.save(prepareEligibilityEntity(hmtAutomationPnrEvent, mealIneligibleReasons));
        log.info("TransactionId: {} ClassName: {} method: {} eligibility: {}", hmtAutomationPnrEvent.getTransactionId(),
                this.getClass().getSimpleName(), "checkEligibilityCriteriaForMealsIssuance", eligibility.toString());

        if (!mealIneligibleReasons.isEmpty()) {
            throw new HMTAutomationException("400", mealIneligibleReasons.toString());
        }
    }

    private boolean isPnrLiesInTopTierCategory(List<Pax> paxList, String transactionId) {
        log.info("TransactionId: {} ClassName: {} method: {} paxList: {}", transactionId,
                this.getClass().getSimpleName(), "isPnrLiesInTopTierCategory", paxList);
        return Objects.nonNull(paxList) && paxList.stream().filter(Objects::nonNull).
                anyMatch(pax -> Stream.of(TierStatusEnum.values()).anyMatch(tierStatusObj -> tierStatusObj.getValue().
                        equalsIgnoreCase(pax.getTierStatus())));
    }

    private boolean isPnrLiesInPremiumCabinCategory(List<Pax> paxList, String transactionId) {
        log.info("TransactionId: {} ClassName: {} method: {} paxList: {}", transactionId,
                this.getClass().getSimpleName(), "isPnrLiesInPremiumCabinCategory", paxList);
        return Objects.nonNull(paxList) && paxList.stream().filter(Objects::nonNull).
                anyMatch(pax -> {
                    Optional<ClassOfServiceMappings> classOfServiceMappings = dbCodesCachingUtil.getClassOfServiceMappings(pax.getClassOfService());
                    return classOfServiceMappings.isPresent() ? CLASS_OF_SERVICE.contains(classOfServiceMappings.get().getClassOfService()) : false;
                });
    }

    private boolean isPnrOfTypeNonRev(List<Pax> paxList, String transactionId) {
        log.info("TransactionId: {} ClassName: {} method: {} paxList: {}", transactionId,
                this.getClass().getSimpleName(), "isPnrOfTypeNonRev", paxList);
        return Objects.isNull(paxList) || paxList.stream().filter(Objects::nonNull).anyMatch(Pax::getIsNonRev);
    }

    private boolean isPnrOfTypeStandBy(List<Pax> paxList, String transactionId){
        log.info("TransactionId: {} ClassName: {} method: {} paxList: {}", transactionId,
                this.getClass().getSimpleName(), "isPnrOfTypeStandBy", paxList);
        return Objects.isNull(paxList) || paxList.stream().filter(Objects::nonNull).anyMatch(Pax::getIsStandBy);
    }

    private boolean isPnrOfTypeNonAADV(List<Pax> paxList, String transactionId){
        log.info("TransactionId: {} ClassName: {} method: {} paxList: {}", transactionId,
                this.getClass().getSimpleName(), "isPnrOfTypeNonAADV", paxList);
        return Objects.isNull(paxList) || paxList.stream().filter(Objects::nonNull).anyMatch(pax -> StringUtils.
                isEmpty(pax.getAdvantageNumber()));
    }

    private Eligibility prepareEligibilityEntity(HMTAutomationPnr hmtAutomationPnrEvent, StringBuilder mealIneligibleReasons) {
        Eligibility eligibility = new Eligibility();
        EligibilityId eligibilityId = new EligibilityId();
        eligibilityId.setFltNbr(hmtAutomationPnrEvent.getFlightKey().getFltNum());
        eligibilityId.setSchdDepCity(hmtAutomationPnrEvent.getFlightKey().getDepSta());
        eligibilityId.setSchdDepDateTime(ApplicationUtils.getDateFromString(hmtAutomationPnrEvent.getFlightKey().getFltOrgDate(),
                ApplicationConstants.INSTANCE.DATE_FORMAT_YYYY_MM_DD));
        eligibilityId.setRecLoc(hmtAutomationPnrEvent.getRecLoc());
        eligibility.setEligibilityId(eligibilityId);
        eligibility.setEligible(mealIneligibleReasons.isEmpty());
        eligibility.setReason(mealIneligibleReasons.toString());
        eligibility.setIssuanceType(hmtAutomationPnrEvent.getOffer().getOfferType().value());
        eligibility.setCreatedTime(new Timestamp(System.currentTimeMillis()));
        eligibility.setLastUpdatedTime(new Timestamp(System.currentTimeMillis()));
        return eligibility;
    }

    private boolean isAutomatedDigitalMealsEnabled(String portOfAccomodation, String transactionId) {
        log.info("TransactionId: {} ClassName: {} method: {} depStation: {}", transactionId,
                this.getClass().getSimpleName(), "isAutomatedDigitalMealsEnabled", portOfAccomodation);
        CountryAirportCode countryAirportCode = dbCodesCachingUtil.findByAirportCode(portOfAccomodation);
        return countryAirportCode.getFeatures().stream().filter(Objects::nonNull).
                anyMatch(feature -> feature.getFeatureName().equals(AUTOMATED_DIGITAL_MEALS));
    }

}




-----------------------------------------------
package com.aa.hmtautomation.eligibility.service.impl;

import com.aa.hmtautomation.DbCodesCachingUtil;
import com.aa.hmtautomation.db.entity.ClassOfServiceMappings;
import com.aa.hmtautomation.db.entity.CountryAirportCode;
import com.aa.hmtautomation.db.entity.Feature;
import com.aa.hmtautomation.db.repository.ClassOfServiceMappingsRepository;
import com.aa.hmtautomation.db.entity.Eligibility;
import com.aa.hmtautomation.db.repository.EligibilityRepository;
import com.aa.hmtautomation.db.repository.OfferDetailsRepository;
import com.aa.hmtautomation.eligibility.model.FlightKey;
import com.aa.hmtautomation.eligibility.model.HMTAutomationPnr;
import com.aa.hmtautomation.eligibility.model.Offer;
import com.aa.hmtautomation.eligibility.model.Pax;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.lenient;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
public class MealEligibilityServiceImplTest {

    @InjectMocks
    private MealEligibilityServiceImpl mealEligibilityService;

    @Mock
    private OfferDetailsRepository offerDetailsRepository;

    @Mock
    private ClassOfServiceMappingsRepository classOfServiceMappingsRepository;

    @Mock
    private EligibilityRepository eligibilityRepository;

    @Mock
    private DbCodesCachingUtil dbCodesCachingUtil;

    @BeforeEach
    public void setUp() {
        lenient().when(offerDetailsRepository.isMealIssuedOnPnr(any(), any(), any(), any())).thenReturn(Boolean.FALSE);
        lenient().when(eligibilityRepository.save(any())).thenReturn(new Eligibility());
    }

    @Test
    public void testProcessHotelOrMealEvent() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setPaxList(paxList);
        ClassOfServiceMappings classOfServiceMappings = new ClassOfServiceMappings();
        classOfServiceMappings.setClassOfService("BUSINESS CLASS");
        classOfServiceMappings.setClassOfServiceCodes("D");
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.getClassOfServiceMappings("D")).thenReturn(Optional.of(classOfServiceMappings));
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        boolean topTierStatus = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrLiesInTopTierCategory", paxList, "test-31276-4544-5656-43345");
        boolean premiumCabinCategory = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrLiesInPremiumCabinCategory",paxList, "test-31276-4544-5656-43345");
        mealEligibilityService.checkEligibility(hmtAutomationPnr);
        assertTrue(topTierStatus);
        assertTrue(premiumCabinCategory);
    }

    @Test
    public void testProcessHotelOrMealEventTierStatusOrCabinException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("test1");
        firstPax.setClassOfService("O");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("AD");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test2");
        secondPax.setClassOfService("N");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("AD");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setPaxList(paxList);
        hmtAutomationPnr.setFlightKey(flightKey);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().contains("Pnr is not eligible for meals as it doesn't lie in top tier or premium cabin category"));
    }

    @Test
    public void testProcessHotelOrMealEventNonRevException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(true);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setPaxList(paxList);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().contains("Pnr is not eligible for meals as it is of type non revenue"));
    }

    @Test
    public void testProcessHotelOrMealEventStandByException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setOriginStation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(true);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setPaxList(paxList);
        hmtAutomationPnr.setFlightKey(flightKey);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().contains("Pnr is not eligible for meals as it is of type standBy"));
    }

    @Test
    public void testProcessHotelOrMealEventAPassException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(true);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setPaxList(paxList);
        hmtAutomationPnr.setFlightKey(flightKey);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().contains("Pnr is not eligible for meals as it is of type apass"));
    }

    @Test
    public void testProcessHotelOrMealEventNonAADVException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setPaxList(paxList);
        hmtAutomationPnr.setFlightKey(flightKey);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().contains("Pnr is not eligible for meals as it is of type NonAADV"));
    }

    @Test
    public void testProcessHotelOrMealEventPortOfAccException() {
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("DFW");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        paxList.add(firstPax);
        paxList.add(secondPax);
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setPaxList(paxList);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().
                contains("Pnr is not eligible for meals as port of accommodation is same as destination station"));
    }

    @Test
    public void testIsPnrOfTypeNonRevWhenTrue(){
        String transactionId = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(true);
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeNonRev", paxList, transactionId);
        assertTrue(result);
    }

    @Test
    public void testIsPnrOfTypeNonRevWhenFalse(){
        String transactionId = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeNonRev", paxList, transactionId);
        assertFalse(result);
    }

    @Test
    public void testIsPnrOfTypeStandByWhenTrue(){
        String transaction = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsStandBy(true);
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsStandBy(false);
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeStandBy", paxList, transaction);
        assertTrue(result);
    }

    @Test
    public void testIsPnrOfTypeStandByWhenFalse(){
        String transaction = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsStandBy(false);
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsStandBy(false);
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeStandBy", paxList, transaction);
        assertFalse(result);
    }

    @Test
    public void testIsPnrOfTypeNonAADVWhenTrue(){
        String transaction = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setAdvantageNumber("");
        Pax secondPax = new Pax();
        secondPax.setClassOfService("V");
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeNonAADV", paxList, transaction);
        assertTrue(result);
    }

    @Test
    public void testIsPnrOfTypeNonAADVWhenFalse(){
        String transaction = "test-31276-4544-5656-43345";
        List<Pax> paxList = new ArrayList<Pax>();
        Pax firstPax = new Pax();
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setClassOfService("V");
        secondPax.setAdvantageNumber("test2");
        paxList.add(firstPax);
        paxList.add(secondPax);
        boolean result = ReflectionTestUtils.invokeMethod(mealEligibilityService, "isPnrOfTypeNonAADV", paxList, transaction);
        assertFalse(result);
    }

    @Test
    public void testMealIssuedOnPnr_whenException(){
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setIsAPass(false);
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        hmtAutomationPnr.setDestinationStation("LAX");
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        Pax firstPax = new Pax();
        firstPax.setTierStatus("EXP");
        firstPax.setClassOfService("D");
        firstPax.setIsNonRev(false);
        firstPax.setIsStandBy(false);
        firstPax.setAdvantageNumber("test");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test");
        secondPax.setClassOfService("V");
        secondPax.setIsNonRev(false);
        secondPax.setIsStandBy(false);
        secondPax.setAdvantageNumber("test");
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setPaxList(List.of(firstPax, secondPax));
        when(offerDetailsRepository.isMealIssuedOnPnr(any(), any(), any(), any())).thenReturn(Boolean.TRUE);
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("AUTOMATED_DIGITAL_MEALS");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        assertTrue(assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr)).getMessage().
                contains("Meal is already issued on pnr"));
    }

    @Test
    public void testAllMealIneligibleReasons_whenException(){
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setIsAPass(true);
        hmtAutomationPnr.setPortOfAccomodation("LAX");
        hmtAutomationPnr.setDestinationStation("DFW");
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        hmtAutomationPnr.setTransactionId("test-31276-4544-5656-43345");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        Pax firstPax = new Pax();
        firstPax.setTierStatus("test1");
        firstPax.setClassOfService("O");
        firstPax.setIsNonRev(true);
        firstPax.setIsStandBy(true);
        firstPax.setAdvantageNumber("");
        Pax secondPax = new Pax();
        secondPax.setTierStatus("test2");
        secondPax.setClassOfService("N");
        secondPax.setIsNonRev(true);
        secondPax.setIsStandBy(true);
        secondPax.setAdvantageNumber("");
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setPaxList(List.of(firstPax, secondPax));
        CountryAirportCode countryAirportCode = new CountryAirportCode();
        List<Feature> featureList = new ArrayList();
        Feature feature = new Feature();
        feature.setFeatureName("UBER_API");
        featureList.add(feature);
        countryAirportCode.setFeatures(featureList);
        when(dbCodesCachingUtil.findByAirportCode(any())).thenReturn(countryAirportCode);
        when(offerDetailsRepository.isMealIssuedOnPnr(any(), any(), any(), any())).thenReturn(Boolean.TRUE);
        Exception exception = assertThrows(Exception.class, () -> mealEligibilityService.
                checkEligibility(hmtAutomationPnr));
        assertTrue(exception.getMessage().contains("Pnr is not eligible for meals as it doesn't lie in top tier or premium cabin category"));
        assertTrue(exception.getMessage().contains("Pnr is not eligible for meals as it is of type apass"));
        assertTrue(exception.getMessage().contains("Meal is already issued on pnr"));
        assertTrue(exception.getMessage().contains("Port of Accommodation " + hmtAutomationPnr.getPortOfAccomodation() + " is not eligible for Automated Digital Meal"));
    }

    @Test
    public void testPrepareEligibilityEntity(){
        HMTAutomationPnr hmtAutomationPnr = new HMTAutomationPnr();
        hmtAutomationPnr.setPortOfAccomodation("DFW");
        FlightKey flightKey = new FlightKey();
        flightKey.setFltNum("2130");
        flightKey.setDepSta("LAX");
        flightKey.setFltOrgDate("2024-04-23");
        Offer offer = new Offer();
        offer.setOfferType(Offer.OfferType.MEALS);
        hmtAutomationPnr.setOffer(offer);
        hmtAutomationPnr.setFlightKey(flightKey);
        hmtAutomationPnr.setRecLoc("XYANHC");
        StringBuilder mealIneligibleReasons = new StringBuilder();
        mealIneligibleReasons.append("Pnr is not eligible for meals as it is of type non revenue.\n");
        mealIneligibleReasons.append("Meal is already issued on pnr.\n");
        Eligibility eligibility = ReflectionTestUtils.invokeMethod(mealEligibilityService, "prepareEligibilityEntity", hmtAutomationPnr,
                mealIneligibleReasons);
        assertTrue(eligibility.getReason().contains("Meal is already issued on pnr.\n"));
        assertEquals(eligibility.getEligibilityId().getRecLoc(), "XYANHC");
    }
}


---------------------------------------
"C:\Program Files\Amazon Corretto\jdk21.0.0_35\bin\java.exe" -ea -Didea.test.cyclic.buffer.size=1048576 "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.3.3\lib\idea_rt.jar=63785:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.3.3\bin" -Dfile.encoding=UTF-8 -Dsun.stdout.encoding=UTF-8 -Dsun.stderr.encoding=UTF-8 -classpath "C:\Users\2134287\.m2\repository\org\junit\platform\junit-platform-launcher\1.10.2\junit-platform-launcher-1.10.2.jar;C:\Users\2134287\.m2\repository\org\junit\vintage\junit-vintage-engine\5.10.2\junit-vintage-engine-5.10.2.jar;C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.3.3\lib\idea_rt.jar;C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.3.3\plugins\junit\lib\junit5-rt.jar;C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.3.3\plugins\junit\lib\junit-rt.jar;C:\Anjali_Bharti_007\10-HMT\3-HMT\hmt-offer-eligibility-service\target\test-classes;C:\Anjali_Bharti_007\10-HMT\3-HMT\hmt-offer-eligibility-service\target\classes;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-web\3.3.0\spring-boot-starter-web-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter\3.3.0\spring-boot-starter-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-logging\3.3.0\spring-boot-starter-logging-3.3.0.jar;C:\Users\2134287\.m2\repository\ch\qos\logback\logback-classic\1.5.6\logback-classic-1.5.6.jar;C:\Users\2134287\.m2\repository\ch\qos\logback\logback-core\1.5.6\logback-core-1.5.6.jar;C:\Users\2134287\.m2\repository\org\apache\logging\log4j\log4j-to-slf4j\2.23.1\log4j-to-slf4j-2.23.1.jar;C:\Users\2134287\.m2\repository\org\apache\logging\log4j\log4j-api\2.23.1\log4j-api-2.23.1.jar;C:\Users\2134287\.m2\repository\org\slf4j\jul-to-slf4j\2.0.13\jul-to-slf4j-2.0.13.jar;C:\Users\2134287\.m2\repository\jakarta\annotation\jakarta.annotation-api\2.1.1\jakarta.annotation-api-2.1.1.jar;C:\Users\2134287\.m2\repository\org\yaml\snakeyaml\2.2\snakeyaml-2.2.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-json\3.3.0\spring-boot-starter-json-3.3.0.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.17.1\jackson-datatype-jdk8-2.17.1.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.17.1\jackson-datatype-jsr310-2.17.1.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.17.1\jackson-module-parameter-names-2.17.1.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-tomcat\3.3.0\spring-boot-starter-tomcat-3.3.0.jar;C:\Users\2134287\.m2\repository\org\apache\tomcat\embed\tomcat-embed-core\10.1.24\tomcat-embed-core-10.1.24.jar;C:\Users\2134287\.m2\repository\org\apache\tomcat\embed\tomcat-embed-el\10.1.24\tomcat-embed-el-10.1.24.jar;C:\Users\2134287\.m2\repository\org\apache\tomcat\embed\tomcat-embed-websocket\10.1.24\tomcat-embed-websocket-10.1.24.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-web\6.1.8\spring-web-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-beans\6.1.8\spring-beans-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-webmvc\6.1.8\spring-webmvc-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-aop\6.1.8\spring-aop-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-context\6.1.8\spring-context-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-expression\6.1.8\spring-expression-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-oauth2-resource-server\3.3.0\spring-boot-starter-oauth2-resource-server-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-config\6.3.0\spring-security-config-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-core\6.3.0\spring-security-core-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-crypto\6.3.0\spring-security-crypto-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-oauth2-resource-server\6.3.0\spring-security-oauth2-resource-server-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-oauth2-core\6.3.0\spring-security-oauth2-core-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-oauth2-jose\6.3.0\spring-security-oauth2-jose-6.3.0.jar;C:\Users\2134287\.m2\repository\com\nimbusds\nimbus-jose-jwt\9.37.3\nimbus-jose-jwt-9.37.3.jar;C:\Users\2134287\.m2\repository\com\github\stephenc\jcip\jcip-annotations\1.0-1\jcip-annotations-1.0-1.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-oauth2-client\3.3.0\spring-boot-starter-oauth2-client-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-oauth2-client\6.3.0\spring-security-oauth2-client-6.3.0.jar;C:\Users\2134287\.m2\repository\com\nimbusds\oauth2-oidc-sdk\9.43.4\oauth2-oidc-sdk-9.43.4.jar;C:\Users\2134287\.m2\repository\com\nimbusds\content-type\2.2\content-type-2.2.jar;C:\Users\2134287\.m2\repository\com\nimbusds\lang-tag\1.7\lang-tag-1.7.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-data-jpa\3.3.0\spring-boot-starter-data-jpa-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-aop\3.3.0\spring-boot-starter-aop-3.3.0.jar;C:\Users\2134287\.m2\repository\org\aspectj\aspectjweaver\1.9.22\aspectjweaver-1.9.22.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-jdbc\3.3.0\spring-boot-starter-jdbc-3.3.0.jar;C:\Users\2134287\.m2\repository\com\zaxxer\HikariCP\5.1.0\HikariCP-5.1.0.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-jdbc\6.1.8\spring-jdbc-6.1.8.jar;C:\Users\2134287\.m2\repository\org\hibernate\orm\hibernate-core\6.5.2.Final\hibernate-core-6.5.2.Final.jar;C:\Users\2134287\.m2\repository\jakarta\persistence\jakarta.persistence-api\3.1.0\jakarta.persistence-api-3.1.0.jar;C:\Users\2134287\.m2\repository\jakarta\transaction\jakarta.transaction-api\2.0.1\jakarta.transaction-api-2.0.1.jar;C:\Users\2134287\.m2\repository\org\jboss\logging\jboss-logging\3.5.3.Final\jboss-logging-3.5.3.Final.jar;C:\Users\2134287\.m2\repository\org\hibernate\common\hibernate-commons-annotations\6.0.6.Final\hibernate-commons-annotations-6.0.6.Final.jar;C:\Users\2134287\.m2\repository\io\smallrye\jandex\3.1.2\jandex-3.1.2.jar;C:\Users\2134287\.m2\repository\com\fasterxml\classmate\1.7.0\classmate-1.7.0.jar;C:\Users\2134287\.m2\repository\net\bytebuddy\byte-buddy\1.14.16\byte-buddy-1.14.16.jar;C:\Users\2134287\.m2\repository\jakarta\inject\jakarta.inject-api\2.0.1\jakarta.inject-api-2.0.1.jar;C:\Users\2134287\.m2\repository\org\antlr\antlr4-runtime\4.13.0\antlr4-runtime-4.13.0.jar;C:\Users\2134287\.m2\repository\org\springframework\data\spring-data-jpa\3.3.0\spring-data-jpa-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\data\spring-data-commons\3.3.0\spring-data-commons-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-orm\6.1.8\spring-orm-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-tx\6.1.8\spring-tx-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-aspects\6.1.8\spring-aspects-6.1.8.jar;C:\Users\2134287\.m2\repository\com\oracle\database\jdbc\ojdbc11\21.9.0.0\ojdbc11-21.9.0.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-cache\3.3.0\spring-boot-starter-cache-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-context-support\6.1.8\spring-context-support-6.1.8.jar;C:\Users\2134287\.m2\repository\org\ehcache\ehcache\3.10.8\ehcache-3.10.8.jar;C:\Users\2134287\.m2\repository\javax\cache\cache-api\1.1.1\cache-api-1.1.1.jar;C:\Users\2134287\.m2\repository\org\slf4j\slf4j-api\2.0.13\slf4j-api-2.0.13.jar;C:\Users\2134287\.m2\repository\org\glassfish\jaxb\jaxb-runtime\4.0.5\jaxb-runtime-4.0.5.jar;C:\Users\2134287\.m2\repository\org\glassfish\jaxb\jaxb-core\4.0.5\jaxb-core-4.0.5.jar;C:\Users\2134287\.m2\repository\org\eclipse\angus\angus-activation\2.0.2\angus-activation-2.0.2.jar;C:\Users\2134287\.m2\repository\org\glassfish\jaxb\txw2\4.0.5\txw2-4.0.5.jar;C:\Users\2134287\.m2\repository\com\sun\istack\istack-commons-runtime\4.1.2\istack-commons-runtime-4.1.2.jar;C:\Users\2134287\.m2\repository\org\apache\kafka\kafka-streams\3.7.0\kafka-streams-3.7.0.jar;C:\Users\2134287\.m2\repository\org\apache\kafka\kafka-clients\3.7.0\kafka-clients-3.7.0.jar;C:\Users\2134287\.m2\repository\com\github\luben\zstd-jni\1.5.5-6\zstd-jni-1.5.5-6.jar;C:\Users\2134287\.m2\repository\org\lz4\lz4-java\1.8.0\lz4-java-1.8.0.jar;C:\Users\2134287\.m2\repository\org\xerial\snappy\snappy-java\1.1.10.5\snappy-java-1.1.10.5.jar;C:\Users\2134287\.m2\repository\org\rocksdb\rocksdbjni\7.9.2\rocksdbjni-7.9.2.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.17.1\jackson-annotations-2.17.1.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.17.1\jackson-databind-2.17.1.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.17.1\jackson-core-2.17.1.jar;C:\Users\2134287\.m2\repository\org\springframework\cloud\spring-cloud-stream-binder-kafka-streams\4.1.1\spring-cloud-stream-binder-kafka-streams-4.1.1.jar;C:\Users\2134287\.m2\repository\org\springframework\cloud\spring-cloud-stream-binder-kafka-core\4.1.1\spring-cloud-stream-binder-kafka-core-4.1.1.jar;C:\Users\2134287\.m2\repository\org\springframework\cloud\spring-cloud-stream\4.1.1\spring-cloud-stream-4.1.1.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-validation\3.3.0\spring-boot-starter-validation-3.3.0.jar;C:\Users\2134287\.m2\repository\org\hibernate\validator\hibernate-validator\8.0.1.Final\hibernate-validator-8.0.1.Final.jar;C:\Users\2134287\.m2\repository\org\springframework\integration\spring-integration-jmx\6.3.0\spring-integration-jmx-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\cloud\spring-cloud-function-context\4.1.1\spring-cloud-function-context-4.1.1.jar;C:\Users\2134287\.m2\repository\net\jodah\typetools\0.6.2\typetools-0.6.2.jar;C:\Users\2134287\.m2\repository\org\springframework\cloud\spring-cloud-function-core\4.1.1\spring-cloud-function-core-4.1.1.jar;C:\Users\2134287\.m2\repository\org\jetbrains\kotlin\kotlin-stdlib-jdk8\1.9.24\kotlin-stdlib-jdk8-1.9.24.jar;C:\Users\2134287\.m2\repository\org\jetbrains\kotlin\kotlin-stdlib\1.9.24\kotlin-stdlib-1.9.24.jar;C:\Users\2134287\.m2\repository\org\jetbrains\annotations\13.0\annotations-13.0.jar;C:\Users\2134287\.m2\repository\org\jetbrains\kotlin\kotlin-stdlib-jdk7\1.9.24\kotlin-stdlib-jdk7-1.9.24.jar;C:\Users\2134287\.m2\repository\org\springframework\kafka\spring-kafka\3.2.0\spring-kafka-3.2.0.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-messaging\6.1.8\spring-messaging-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\retry\spring-retry\2.0.6\spring-retry-2.0.6.jar;C:\Users\2134287\.m2\repository\org\springframework\integration\spring-integration-kafka\6.3.0\spring-integration-kafka-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\integration\spring-integration-core\6.3.0\spring-integration-core-6.3.0.jar;C:\Users\2134287\.m2\repository\io\projectreactor\reactor-core\3.6.6\reactor-core-3.6.6.jar;C:\Users\2134287\.m2\repository\org\reactivestreams\reactive-streams\1.0.4\reactive-streams-1.0.4.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-test\3.3.0\spring-boot-starter-test-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-test\3.3.0\spring-boot-test-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-test-autoconfigure\3.3.0\spring-boot-test-autoconfigure-3.3.0.jar;C:\Users\2134287\.m2\repository\com\jayway\jsonpath\json-path\2.9.0\json-path-2.9.0.jar;C:\Users\2134287\.m2\repository\jakarta\xml\bind\jakarta.xml.bind-api\4.0.2\jakarta.xml.bind-api-4.0.2.jar;C:\Users\2134287\.m2\repository\jakarta\activation\jakarta.activation-api\2.1.3\jakarta.activation-api-2.1.3.jar;C:\Users\2134287\.m2\repository\net\minidev\json-smart\2.5.1\json-smart-2.5.1.jar;C:\Users\2134287\.m2\repository\net\minidev\accessors-smart\2.5.1\accessors-smart-2.5.1.jar;C:\Users\2134287\.m2\repository\org\ow2\asm\asm\9.6\asm-9.6.jar;C:\Users\2134287\.m2\repository\org\assertj\assertj-core\3.25.3\assertj-core-3.25.3.jar;C:\Users\2134287\.m2\repository\org\awaitility\awaitility\4.2.1\awaitility-4.2.1.jar;C:\Users\2134287\.m2\repository\org\hamcrest\hamcrest\2.2\hamcrest-2.2.jar;C:\Users\2134287\.m2\repository\org\junit\jupiter\junit-jupiter\5.10.2\junit-jupiter-5.10.2.jar;C:\Users\2134287\.m2\repository\org\junit\jupiter\junit-jupiter-api\5.10.2\junit-jupiter-api-5.10.2.jar;C:\Users\2134287\.m2\repository\org\opentest4j\opentest4j\1.3.0\opentest4j-1.3.0.jar;C:\Users\2134287\.m2\repository\org\junit\platform\junit-platform-commons\1.10.2\junit-platform-commons-1.10.2.jar;C:\Users\2134287\.m2\repository\org\apiguardian\apiguardian-api\1.1.2\apiguardian-api-1.1.2.jar;C:\Users\2134287\.m2\repository\org\junit\jupiter\junit-jupiter-params\5.10.2\junit-jupiter-params-5.10.2.jar;C:\Users\2134287\.m2\repository\org\junit\jupiter\junit-jupiter-engine\5.10.2\junit-jupiter-engine-5.10.2.jar;C:\Users\2134287\.m2\repository\org\junit\platform\junit-platform-engine\1.10.2\junit-platform-engine-1.10.2.jar;C:\Users\2134287\.m2\repository\org\mockito\mockito-core\5.11.0\mockito-core-5.11.0.jar;C:\Users\2134287\.m2\repository\net\bytebuddy\byte-buddy-agent\1.14.16\byte-buddy-agent-1.14.16.jar;C:\Users\2134287\.m2\repository\org\objenesis\objenesis\3.3\objenesis-3.3.jar;C:\Users\2134287\.m2\repository\org\mockito\mockito-junit-jupiter\5.11.0\mockito-junit-jupiter-5.11.0.jar;C:\Users\2134287\.m2\repository\org\skyscreamer\jsonassert\1.5.1\jsonassert-1.5.1.jar;C:\Users\2134287\.m2\repository\com\vaadin\external\google\android-json\0.0.20131108.vaadin1\android-json-0.0.20131108.vaadin1.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-core\6.1.8\spring-core-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-jcl\6.1.8\spring-jcl-6.1.8.jar;C:\Users\2134287\.m2\repository\org\springframework\spring-test\6.1.8\spring-test-6.1.8.jar;C:\Users\2134287\.m2\repository\org\xmlunit\xmlunit-core\2.9.1\xmlunit-core-2.9.1.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-test\6.3.0\spring-security-test-6.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\security\spring-security-web\6.3.0\spring-security-web-6.3.0.jar;C:\Users\2134287\.m2\repository\junit\junit\4.13.2\junit-4.13.2.jar;C:\Users\2134287\.m2\repository\org\hamcrest\hamcrest-core\2.2\hamcrest-core-2.2.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-starter-actuator\3.3.0\spring-boot-starter-actuator-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-actuator-autoconfigure\3.3.0\spring-boot-actuator-autoconfigure-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-actuator\3.3.0\spring-boot-actuator-3.3.0.jar;C:\Users\2134287\.m2\repository\io\micrometer\micrometer-observation\1.13.0\micrometer-observation-1.13.0.jar;C:\Users\2134287\.m2\repository\io\micrometer\micrometer-commons\1.13.0\micrometer-commons-1.13.0.jar;C:\Users\2134287\.m2\repository\io\micrometer\micrometer-jakarta9\1.13.0\micrometer-jakarta9-1.13.0.jar;C:\Users\2134287\.m2\repository\io\micrometer\micrometer-core\1.13.0\micrometer-core-1.13.0.jar;C:\Users\2134287\.m2\repository\org\hdrhistogram\HdrHistogram\2.2.1\HdrHistogram-2.2.1.jar;C:\Users\2134287\.m2\repository\org\latencyutils\LatencyUtils\2.0.3\LatencyUtils-2.0.3.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-devtools\3.3.0\spring-boot-devtools-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot\3.3.0\spring-boot-3.3.0.jar;C:\Users\2134287\.m2\repository\org\springframework\boot\spring-boot-autoconfigure\3.3.0\spring-boot-autoconfigure-3.3.0.jar;C:\Users\2134287\.m2\repository\org\projectlombok\lombok\1.18.32\lombok-1.18.32.jar;C:\Users\2134287\.m2\repository\org\springdoc\springdoc-openapi-starter-webmvc-ui\2.5.0\springdoc-openapi-starter-webmvc-ui-2.5.0.jar;C:\Users\2134287\.m2\repository\org\springdoc\springdoc-openapi-starter-webmvc-api\2.5.0\springdoc-openapi-starter-webmvc-api-2.5.0.jar;C:\Users\2134287\.m2\repository\org\springdoc\springdoc-openapi-starter-common\2.5.0\springdoc-openapi-starter-common-2.5.0.jar;C:\Users\2134287\.m2\repository\io\swagger\core\v3\swagger-core-jakarta\2.2.21\swagger-core-jakarta-2.2.21.jar;C:\Users\2134287\.m2\repository\org\apache\commons\commons-lang3\3.14.0\commons-lang3-3.14.0.jar;C:\Users\2134287\.m2\repository\io\swagger\core\v3\swagger-annotations-jakarta\2.2.21\swagger-annotations-jakarta-2.2.21.jar;C:\Users\2134287\.m2\repository\io\swagger\core\v3\swagger-models-jakarta\2.2.21\swagger-models-jakarta-2.2.21.jar;C:\Users\2134287\.m2\repository\jakarta\validation\jakarta.validation-api\3.0.2\jakarta.validation-api-3.0.2.jar;C:\Users\2134287\.m2\repository\com\fasterxml\jackson\dataformat\jackson-dataformat-yaml\2.17.1\jackson-dataformat-yaml-2.17.1.jar;C:\Users\2134287\.m2\repository\org\webjars\swagger-ui\5.13.0\swagger-ui-5.13.0.jar" com.intellij.rt.junit.JUnitStarter -ideVersion5 -junit5 com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImplTest
WARNING: A Java agent has been loaded dynamically (C:\Users\2134287\.m2\repository\net\bytebuddy\byte-buddy-agent\1.14.16\byte-buddy-agent-1.14.16.jar)
WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
WARNING: Dynamic loading of agents will be disallowed by default in a future release
OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
13:59:16.907 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@4f67e3df[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@56681eaf[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@72d0f2b4[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=DFW,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.912 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@4f67e3df[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@56681eaf[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@72d0f2b4[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=DFW,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.912 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.913 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.915 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.926 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.927 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@6d2dc9d2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@1da4b6b3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.933 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:16.948 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:16.962 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@440eaa07[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@7fc7c4a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@7aa9e414[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@440eaa07[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@7fc7c4a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@7aa9e414[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:16.965 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}

org.opentest4j.AssertionFailedError: Expected java.lang.Exception to be thrown, but nothing was thrown.

	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:152)
	at org.junit.jupiter.api.AssertThrows.assertThrows(AssertThrows.java:73)
	at org.junit.jupiter.api.AssertThrows.assertThrows(AssertThrows.java:35)
	at org.junit.jupiter.api.Assertions.assertThrows(Assertions.java:3115)
	at com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImplTest.testProcessHotelOrMealEventNonAADVException(MealEligibilityServiceImplTest.java:310)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

13:59:17.002 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@548d5ed3[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@21c7208d[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.013 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@7f7af971[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=<null>,tierStatus=<null>,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@23382f76[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=<null>,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.024 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@5a82ebf8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68fe48d7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.031 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@5918c260[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=<null>,tierStatus=<null>,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@3d7b1f1c[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=<null>,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test2,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@7ed9499e[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@28e19366[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@5b275174[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=<null>,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@7ed9499e[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@28e19366[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@5b275174[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=<null>,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.040 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@10ef5fa0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@244e619a[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=AD,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.041 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.041 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.050 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@571a9686[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@719d35e8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=<null>,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.060 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@188b6035[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@4a34e9f[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@6f6621e3[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.060 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@188b6035[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@4a34e9f[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@6f6621e3[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.060 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.061 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.061 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.062 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.062 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@3fc05ea2[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7c891ba7[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.062 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.063 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2a03d65c[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6642dc5a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@43da41e[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=LAX,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2a03d65c[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6642dc5a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@43da41e[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=LAX,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.073 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: LAX
13:59:17.073 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}

org.opentest4j.AssertionFailedError: 
Expected :true
Actual   :false
<Click to see difference>


	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)
	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)
	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)
	at org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)
	at com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImplTest.testAllMealIneligibleReasons_whenException(MealEligibilityServiceImplTest.java:537)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

13:59:17.080 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2acbc859[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6ab7ce48[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@2c6aed22[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.080 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2acbc859[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6ab7ce48[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@2c6aed22[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.080 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.080 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.081 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.081 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.081 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@e322ec9[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@7acfb656[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.081 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.082 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.090 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@52cb4f50[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@25a5c7db[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@4d27d9d[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.091 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@52cb4f50[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@25a5c7db[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@4d27d9d[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.091 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.091 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.091 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.092 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.092 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@28f878a0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@20411320[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.092 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.093 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.100 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@12abdfb[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@b0e5507[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@6bbe50c9[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.100 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@12abdfb[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@b0e5507[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@6bbe50c9[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.100 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.100 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.101 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.101 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.101 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@3c46dcbe[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@68577ba8[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.101 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.101 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.110 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.111 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.112 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@1c6c6f24[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@2eb917d0[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@c6b2dd9[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=<null>,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.112 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@1c6c6f24[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@2eb917d0[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@c6b2dd9[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=<null>,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.112 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.112 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.113 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.113 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.113 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@43f0c2d1[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@5fb65013[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=test,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.113 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:17.114 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: null ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}
13:59:17.123 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@12ffd1de[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@3d278b4d[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=<null>,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]

Process finished with exit code -1


----------------------------
Test case 2 :
13:59:16.962 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@440eaa07[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@7fc7c4a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@7aa9e414[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@440eaa07[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@7fc7c4a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@7aa9e414[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=false,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=DFW,originStation=<null>,destinationStation=LAX,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.963 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@53a5e217[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=D,tierStatus=EXP,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@624a24f6[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=V,tierStatus=test,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=false,isStandBy=false,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=<null>,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:16.964 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: DFW
13:59:16.965 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}

org.opentest4j.AssertionFailedError: Expected java.lang.Exception to be thrown, but nothing was thrown.

	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:152)
	at org.junit.jupiter.api.AssertThrows.assertThrows(AssertThrows.java:73)
	at org.junit.jupiter.api.AssertThrows.assertThrows(AssertThrows.java:35)
	at org.junit.jupiter.api.Assertions.assertThrows(Assertions.java:3115)
	at com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImplTest.testProcessHotelOrMealEventNonAADVException(MealEligibilityServiceImplTest.java:310)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

////////////////////////////////////////////////////////////
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: processHotelOrMealEvent hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2a03d65c[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6642dc5a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@43da41e[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=LAX,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance hmtAutomationPnrEvent: com.aa.hmtautomation.eligibility.model.HMTAutomationPnr@2a03d65c[flightKey=com.aa.hmtautomation.eligibility.model.FlightKey@6642dc5a[fltNum=2130,fltOrgDate=2024-04-23,depSta=LAX,schDepDateTime=<null>,disruptDepDateTime=<null>],currentSegment=<null>,offer=com.aa.hmtautomation.eligibility.model.Offer@43da41e[offerType=MEALS,noOfNights=<null>,additionalProperties={}],paxList=[com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]],isAPass=true,recLoc=<null>,transactionId=test-31276-4544-5656-43345,portOfAccomodation=LAX,originStation=<null>,destinationStation=DFW,version=<null>,currencyCode=<null>,mealAmount=<null>,flifoCode=<null>,dsrtFltStatus=<null>]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInTopTierCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.071 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrLiesInPremiumCabinCategory paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonAADV paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeNonRev paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.072 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isPnrOfTypeStandBy paxList: [com.aa.hmtautomation.eligibility.model.Pax@148c7c4b[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=O,tierStatus=test1,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]], com.aa.hmtautomation.eligibility.model.Pax@2009f9b0[seqNbr=<null>,firstName=<null>,lastName=<null>,classOfService=N,tierStatus=test2,isWheelChair=<null>,isUM=<null>,isMinor=<null>,isEC261=<null>,isNonRev=true,isStandBy=true,hasPet=<null>,hasServicePet=<null>,isSpecialNeed=<null>,cupId=<null>,advantageNumber=,isElite=<null>,credt=<null>,sSRcodes=[]]]
13:59:17.073 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: isAutomatedDigitalMealsEnabled depStation: LAX
13:59:17.073 [main] INFO com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImpl -- TransactionId: test-31276-4544-5656-43345 ClassName: MealEligibilityServiceImpl method: checkEligibilityCriteriaForMealsIssuance eligibility: Eligibility{eligibilityId=null, issuanceType='null', isEligible=null, reason='null', createdTime=null, lastUpdatedTime=null}

org.opentest4j.AssertionFailedError: 
Expected :true
Actual   :false
<Click to see difference>


	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertTrue.failNotTrue(AssertTrue.java:63)
	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:36)
	at org.junit.jupiter.api.AssertTrue.assertTrue(AssertTrue.java:31)
	at org.junit.jupiter.api.Assertions.assertTrue(Assertions.java:183)
	at com.aa.hmtautomation.eligibility.service.impl.MealEligibilityServiceImplTest.testAllMealIneligibleReasons_whenException(MealEligibilityServiceImplTest.java:537)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)


